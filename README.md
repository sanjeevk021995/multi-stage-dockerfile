# Multi-Stage Dockerfile and Best Practices

## Why Use Multi-Stage Builds?
Multi-stage builds allow you to:
- **Reduce image size**: Only copy the final JAR into a slim runtime image, leaving behind build tools like Maven and JDK.
- **Improve security**: The runtime image doesn’t include compilers or unnecessary utilities, reducing attack surface.
- **Optimize caching**: Dependencies are cached separately from source code, speeding up rebuilds.
- **Separate concerns**: Cleanly divide build environment (Maven + JDK) from runtime environment (JRE).

---

## When to Use Multi-Stage Builds
- **Production deployments**: When you want small, efficient, and secure images.
- **CI/CD pipelines**: To ensure reproducible builds and faster pipelines.
- **Microservices**: Each service can have its own lean image, reducing resource usage.
- **Cloud/Kubernetes environments**: Smaller images pull faster and scale more efficiently.

---

##  Best Practices
- **Use builder stage with full JDK + Maven**  
  Example: `FROM maven:3.9.6-eclipse-temurin-17 AS builder`
- **Use runtime stage with JRE only**  
  Example: `FROM eclipse-temurin:17-jre`
- **Leverage `.dockerignore`** to avoid copying unnecessary files (like `target/` or local configs).
- **Pin versions** of Maven, JDK, and dependencies for reproducibility.
- **Use exec form for ENTRYPOINT**  
  Example: `ENTRYPOINT ["java", "-jar", "app.jar"]`
- **Keep layers minimal**: Combine related commands with `&&` to reduce image size.
- **Run as non-root user** in production for better security.
- **Tag images properly** (avoid `:latest` in production, use semantic versioning).

---

## Example Workflow
1. **Build Stage**  
   - Compiles the Spring Boot app with Maven.  
   - Produces a JAR file in `/build/target`.

2. **Runtime Stage**  
   - Copies only the JAR into a slim JRE image.  
   - Runs the app with `java -jar app.jar`.

---

##  Benefits
- Your **Spring Boot app** runs in a lightweight container.  
- Faster startup and deployment in Kubernetes.  
- Easier scaling since images are smaller and more secure.  
- Clear separation between build and runtime environments.

---

# Dockerfile
```
# Stage 1: Build
FROM maven:3.9.6-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-ofline
COPY . .
RUN mvn clean package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```
##  Dockerfile Breakdown

- **Stage 1 (Builder)**  
  - Uses Maven with JDK 17 to compile the app.  
  - Downloads dependencies (`mvn dependency:go-offline`).  
  - Packages the app into a JAR (`mvn clean package -DskipTests`).  

- **Stage 2 (Runtime)**  
  - Uses a lightweight JRE image.  
  - Copies only the JAR file from the builder stage.  
  - Runs the app with `java -jar app.jar`.  

---
##  Dockerfile Explanation
```dockerfile
# Stage 1: Build
FROM maven:3.9.6-eclipse-temurin-17 AS builder
```
- Uses the official Maven image with JDK 17 (Temurin distribution).
- Names this stage `builder` so you can reference it later in a multi-stage build.

```dockerfile
WORKDIR /build
```
- Sets the working directory inside the container to `/build`.

```dockerfile
COPY pom.xml .
```
- Copies only the `pom.xml` file first.
- This allows Docker to cache dependency downloads separately from source code changes.

```dockerfile
RUN mvn dependency:go-offline
```
- Downloads all Maven dependencies defined in `pom.xml`.
- Ensures faster builds since dependencies don’t need to be re-fetched unless `pom.xml` changes.


```dockerfile
COPY . .
```
- Copies the rest of your project source code into the container.

```dockerfile
RUN mvn clean package -DskipTests
```
- Cleans previous builds and packages the application into a JAR file.
- Skips tests to speed up the build process.

---

```dockerfile
# Stage 2: Runtime
FROM eclipse-temurin:17-jre
```
- Uses a lightweight JRE-only image (no compiler, just runtime).
- Keeps the final image small and secure.

```dockerfile
WORKDIR /app
```
- Sets the working directory inside the runtime container to `/app`.

```dockerfile
COPY --from=builder /build/target/*.jar app.jar
```
- Copies the JAR file built in Stage 1 into the runtime image.
- Renames it to `app.jar`.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```
- Defines the command that always runs when the container starts.
- Uses **exec form** so signals are passed directly to the Java process.
- Runs your Spring Boot application.

---

## Benefits of Multi-Stage Build
- Smaller final image size.  
- Cleaner separation between build tools and runtime environment.  
- Faster builds thanks to dependency caching.  

---
#  Build and Run with Docker

## Build the Image (with Semantic Versioning)
```bash
# Example: version 1.0.0
docker build -t springboot-helloworld:1.0.0 .
```

- Always tag images with a semantic version (`MAJOR.MINOR.PATCH`) instead of `latest`.
- Example progression:
  - `springboot-helloworld:1.0.0`
  - `springboot-helloworld:1.1.0`
  - `springboot-helloworld:2.0.0`

## Run the Container
```bash
docker run -d -p 8080:8080 springboot-helloworld:1.0.0
```

The app will be available at:  
 `http://localhost:8080`

---
# Best Practices Recap
- Use **semantic versioning** for Docker images (`1.0.0`, `1.1.0`, etc.).
- Add `.dockerignore` to avoid copying unnecessary files into the image.
- Add `.gitignore` to keep your repo clean from build artifacts and IDE clutter.
- Tag images properly (avoid `:latest` in production).
- Run containers with explicit port mappings for clarity.
---
