# Experiment: Maven + JUnit + GitHub Actions + Docker Hub

## 1. Objective

Create a Java Maven project, write JUnit tests, push the project to GitHub, and use GitHub Actions to run the tests and automatically publish a Docker image to Docker Hub after the tests pass.

**Publishing is automated.** Do not run manual `docker login`, `docker tag`, or `docker push` commands. Create a Docker Hub access token once and store it as a GitHub repository secret.

## 2. Requirements

- JDK 17
- Eclipse IDE for Java Developers with Maven support
- Git and GitHub account
- Docker Hub account
- Docker Desktop for optional local container testing

Check the installed tools in PowerShell:

```powershell
java -version
git --version
docker --version
```

## 3. Create the Maven project in Eclipse

1. Open Eclipse IDE.
2. Select **File → New → Maven Project**. If it is not visible, select **File → New → Other → Maven → Maven Project**.
3. Use the Maven quickstart archetype, or create a simple project and add the source folders manually.
4. Enter:

| Field | Value |
|---|---|
| Group Id | `com.example` |
| Artifact Id | `maven-github-demo` |
| Version | `1.0-SNAPSHOT` |
| Packaging | `jar` |

5. Click **Finish** and wait for Eclipse to create the project.

## 4. Configure `pom.xml`

Replace the generated POM contents with:

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

Save the file. Right-click the project → **Maven → Update Project** → select the project → **OK**.

## 5. Create the Java application

Under `src/main/java`, create package `com.example`. Inside it, create `GradeCalculator.java`:

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

Under `src/test/java`, create package `com.example`. Inside it, create `GradeCalculatorTest.java`:

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

## 7. Run tests manually in Eclipse

You do not need to type the `mvn` command in Command Prompt or PowerShell. Run the Maven goals through Eclipse.

### A. Run `clean test`

1. In **Package Explorer**, right-click the project `maven-github-demo`.
2. Select **Run As → Maven build...** (not just **Maven test**).
3. In the **Goals** field, enter:

   ```text
   clean test
   ```

4. Click **Apply**, then **Run**.
5. Check the **Console** tab.

**Expected result:** Maven displays `BUILD SUCCESS`, and all four JUnit tests pass.

### B. Run `clean package`

1. Right-click the project again.
2. Select **Run As → Maven build...**.
3. In the **Goals** field, enter:

   ```text
   clean package
   ```

4. Click **Apply**, then **Run**.
5. Check the **Console** tab.

**Expected result:** Maven displays `BUILD SUCCESS`. A JAR file is generated in the project's `target` folder.

**Meaning of the goals:**

- `clean` removes previous build output from the `target` folder.
- `test` compiles the code and runs the JUnit tests.
- `package` runs the earlier build phases, including tests, and packages the application as a JAR.

If **Maven build...** is unavailable, confirm that you installed **Eclipse IDE for Java Developers** with Maven integration. If the project is not recognized as Maven, right-click it and choose **Configure → Convert to Maven Project**.

## 8. Push the project to GitHub

1. Sign in to GitHub and create an empty repository named `maven-github-demo`.
2. Open a terminal in the project root (the directory containing `pom.xml`).
3. Run:

```powershell
git init
git add .
git commit -m "Create Maven JUnit project"
git branch -M main
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/maven-github-demo.git
git push -u origin main
```

Replace `YOUR-GITHUB-USERNAME` with your GitHub username.

## 9. Add the GitHub Actions workflow

Create the folders `.github/workflows` in the project root. Inside `workflows`, create `ci.yml`:

```yaml
name: Maven Test and Docker Hub Publish

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

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

`needs: maven-test` makes publishing wait for successful tests. Pull requests run tests but do not publish an image. GitHub Actions performs Docker Hub authentication using the repository secrets; do not run `docker login` manually.

## 10. Create the Dockerfile

Create `Dockerfile` in the project root, at the same level as `pom.xml`:

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

The GitHub Actions test job runs the JUnit tests first. The Docker build then compiles/packages the application and creates a runtime image. The image runs `GradeCalculator` when started.

## 11. Create a Docker Hub repository and access token

1. Sign in to Docker Hub.
2. Create a repository named `maven-github-demo`.
3. Open **Account Settings → Personal access tokens** (menu wording may vary).
4. Generate a token with **Read & Write** permission.
5. Copy the token and store it securely.

Never put the token in source code, the workflow YAML, screenshots or Git history. If it is exposed, revoke it and generate a replacement.

## 12. Add GitHub repository secrets

1. Open the GitHub repository.
2. Select **Settings → Secrets and variables → Actions**.
3. Click **New repository secret**.
4. Add:

| Secret name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Your Docker Hub Read & Write access token |

Do not hard-code these values in the workflow.

## 13. Commit and push the Dockerfile and workflow

From the project root, run:

```powershell
git add Dockerfile .github/workflows/ci.yml
git commit -m "Add automated Docker Hub publishing"
git push origin main
```

## 14. Verify the pipeline

1. Open GitHub → your repository → **Actions**.
2. Open the latest `Maven Test and Docker Hub Publish` run.
3. Confirm `maven-test` succeeds.
4. Confirm `docker-publish` runs afterwards and succeeds.
5. Open Docker Hub → `maven-github-demo` → **Tags**.
6. Confirm the `latest` tag and commit-SHA tag are present.

Optional: test the published image on a computer with Docker installed:

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

Replace `YOUR-DOCKERHUB-USERNAME` with your Docker Hub username.

## 15. Final project structure

```text
maven-github-demo/
├── .github/
│   └── workflows/
│       └── ci.yml
├── src/
│   ├── main/
│   │   └── java/com/example/GradeCalculator.java
│   └── test/
│       └── java/com/example/GradeCalculatorTest.java
├── Dockerfile
└── pom.xml
```

## Pipeline summary

```text
Create Maven project in Eclipse
        ↓
Write Java application and JUnit tests
        ↓
Run clean test in Eclipse using Maven build
        ↓
Run clean package in Eclipse using Maven build
        ↓
Push project to GitHub
        ↓
GitHub Actions runs Maven tests
        ↓
If tests pass on push to main, build Docker image
        ↓
Authenticate using GitHub repository secrets
        ↓
Push image to Docker Hub
        ↓
Verify image tags in Docker Hub
```

**No manual Docker Hub publishing commands and no Kubernetes are included.**
docker pull YOUR-DOCKERHUB-USERNAME/selenium-testng-automation:latest
docker run --rm YOUR-DOCKERHUB-USERNAME/selenium-testng-automation:latest
