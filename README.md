# Experiment: Maven + JUnit + GitHub Actions + Docker Hub

## 1. Objective

Create a Java Maven project, write JUnit tests, push the project to GitHub, and use GitHub Actions to run the tests and automatically publish a Docker image to Docker Hub after the tests pass.

**Publishing is automated.** Do not run manual `docker login`, `docker tag`, or `docker push` commands. You create a Docker Hub access token once and store it as a GitHub repository secret.

## 2. Requirements

- JDK 17
- Eclipse IDE with Maven support (or another Java IDE)
- Git and GitHub account
- Docker Hub account
- Docker Desktop for optional local container testing

Check in PowerShell:

```powershell
java -version
git --version
docker --version
```

## 3. Create the Maven project

In Eclipse, choose **File → New → Maven Project**. Select `maven-archetype-quickstart` version `1.5`.

Use:

| Field | Value |
|---|---|
| Group Id | `com.example` |
| Artifact Id | `maven-github-demo` |
| Version | `1.0-SNAPSHOT` |
| Package | `com.example` |

## 4. Configure `pom.xml`

Replace the generated POM with:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>maven-github-demo</artifactId>
  <version>1.0-SNAPSHOT</version>
  <properties>
    <maven.compiler.release>17</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <junit.version>5.10.2</junit.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>${junit.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
        <configuration><release>17</release></configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
      </plugin>
    </plugins>
  </build>
</project>
```

## 5. Create the Java application

Create `src/main/java/com/example/GradeCalculator.java`:

```java
package com.example;

public class GradeCalculator {
    public int total(int a, int b, int c) {
        return a + b + c;
    }

    public double average(int a, int b, int c) {
        return total(a, b, c) / 3.0;
    }

    public String result(double average) {
        return average >= 40 ? "PASS" : "FAIL";
    }

    public static void main(String[] args) {
        GradeCalculator g = new GradeCalculator();
        int total = g.total(70, 80, 75);
        double average = g.average(70, 80, 75);
        System.out.println("Total   : " + total);
        System.out.println("Average : " + average);
        System.out.println("Result  : " + g.result(average));
    }
}
```

## 6. Create JUnit tests

Create `src/test/java/com/example/GradeCalculatorTest.java`:

```java
package com.example;

import static org.junit.jupiter.api.Assertions.assertEquals;
import org.junit.jupiter.api.Test;

class GradeCalculatorTest {
    private final GradeCalculator calculator = new GradeCalculator();

    @Test
    void totalTest() {
        assertEquals(225, calculator.total(70, 80, 75));
    }

    @Test
    void averageTest() {
        assertEquals(75.0, calculator.average(70, 80, 75), 0.001);
    }

    @Test
    void passTest() {
        assertEquals("PASS", calculator.result(75.0));
    }

    @Test
    void failTest() {
        assertEquals("FAIL", calculator.result(35.0));
    }
}
```

## 7. Test locally

From the project root (the directory containing `pom.xml`):

```powershell
mvn clean test
mvn clean package
```

Proceed only when Maven reports `BUILD SUCCESS` and all four tests pass.

## 8. Push the project to GitHub

Create a GitHub repository named `maven-github-demo`. Then run from the project root:

```powershell
git init
git add .
git commit -m "Create Maven JUnit project"
git branch -M main
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/maven-github-demo.git
git push -u origin main
```

Replace the username placeholder with your GitHub username.

## 9. Add GitHub Actions workflow

Create `.github/workflows/ci.yml`:

```yaml
name: Maven Test and Docker Hub Publish

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  maven-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      - name: Run JUnit tests
        run: mvn --batch-mode clean verify

  docker-publish:
    needs: maven-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/maven-github-demo:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/maven-github-demo:${{ github.sha }}
```

`needs: maven-test` makes publishing wait for successful tests. Pull requests run tests but do not publish an image.

## 10. Create the Dockerfile

Create `Dockerfile` in the project root:

```dockerfile
FROM maven:3.9.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn --batch-mode clean package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /app/target/classes ./classes
CMD ["java", "-cp", "classes", "com.example.GradeCalculator"]
```

The JUnit tests are run in the first GitHub Actions job. The Docker build compiles/packages the app and creates a runtime image.

## 11. Create Docker Hub repository and access token

1. Sign in to Docker Hub.
2. Create a repository named `maven-github-demo`.
3. Open **Account Settings → Security → Personal access tokens**.
4. Generate a token with **Read & Write** permission.
5. Copy it and store it securely.

Never put the token in source code, YAML, screenshots, or Git history. Revoke it and generate a replacement if exposed.

## 12. Add GitHub repository secrets

Open the GitHub repository → **Settings → Secrets and variables → Actions → New repository secret**.

Create:

| Secret name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Your Docker Hub Read & Write access token |

Do not hard-code these values in the workflow.

## 13. Commit and push the Dockerfile and workflow

```powershell
git add Dockerfile .github/workflows/ci.yml
git commit -m "Add automated Docker Hub publishing"
git push origin main
```

## 14. Verify the pipeline

1. Open GitHub → repository → **Actions**.
2. Open the latest `Maven Test and Docker Hub Publish` run.
3. Confirm `maven-test` succeeds.
4. Confirm `docker-publish` runs afterwards and succeeds.
5. Open Docker Hub → `maven-github-demo` → **Tags**.
6. Confirm `latest` and the commit-SHA tag are present.

Optional: pull and run the published image on a computer with Docker:

```powershell
docker pull YOUR-DOCKERHUB-USERNAME/maven-github-demo:latest
docker run --rm YOUR-DOCKERHUB-USERNAME/maven-github-demo:latest
```

Expected output:

```text
Total   : 225
Average : 75.0
Result  : PASS
```

## 15. Final project structure

```text
maven-github-demo/
├── .github/workflows/ci.yml
├── src/
│   ├── main/java/com/example/GradeCalculator.java
│   └── test/java/com/example/GradeCalculatorTest.java
├── Dockerfile
└── pom.xml
```

## Pipeline summary

```text
Create Maven project → Write JUnit tests → Test locally
→ Push to GitHub → GitHub Actions runs Maven tests
→ If tests pass, build Docker image
→ Authenticate using GitHub secrets
→ Push image to Docker Hub → Verify tags
```

No manual Docker Hub publishing commands and no Kubernetes are included.
