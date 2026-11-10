# Experiment 3 + Docker: Maven, JUnit 5, Docker and Docker Hub

## What this pipeline does

This README follows the supplied **Experiment 3 -- Automated Build and
Dependency Management Using Apache Maven** and extends its generated
Java artifact into a Docker image using the Docker Hub procedure from
**Experiment 4 -- Application Containerization Using Docker**.

**Pipeline:** Eclipse Maven project → Student Grade Calculator → JUnit 5
→ `mvn clean package` → JAR/classes → Docker image → Docker Hub access
token → `docker login` → `docker tag` → `docker push` → `docker pull` on
another computer → `docker run`.

> Scope: No Kubernetes is used. Experiment 3 itself ends at the JAR and
> GitHub Actions Maven build. The Docker steps below are the
> containerization extension needed to continue that project to Docker
> Hub. The source Experiment 4 uses Tomcat for a static HTML
> application; this Maven Java application is not a Tomcat web
> application, so its Dockerfile uses a Java runtime instead.

## 1. Prerequisites

-   Windows 10/11
-   Eclipse IDE for Java Developers with Maven (m2e)
-   JDK 17 or later
-   Git
-   Docker Desktop, installed and running
-   GitHub account
-   Docker Hub account
-   Internet connection

Open PowerShell and verify:

``` powershell
java -version
git --version
docker --version
docker info
```

If `docker info` cannot connect to the daemon, start Docker Desktop and
wait until the engine is running.

## 2. Create the Maven project in Eclipse

1.  Open Eclipse and choose your workspace.
2.  Select **File → New → Maven Project**. If it is not listed, choose
    **File → New → Other → Maven → Maven Project**.
3.  Select the `maven-archetype-quickstart` archetype, version `1.5`.
4.  Enter these coordinates:

  Field         Value
  ------------- ---------------------
  Group Id      `com.example`
  Artifact Id   `maven-github-demo`
  Version       `1.0-SNAPSHOT`
  Package       `com.example`

5.  Click **Finish**. If prompted, type `Y`.
6.  The generated `App.java` and `AppTest.java` are starter files.
    Replace them as described below.

Expected initial Maven layout:

``` text
maven-github-demo/
├── src/
│   ├── main/java/com/example/
│   └── test/java/com/example/
└── pom.xml
```

## 3. Configure `pom.xml` for Java 17 and JUnit 5

Open `pom.xml`. Retain the generated project coordinates and make sure
the properties, dependency and Surefire plugin below are present. Remove
the archetype's old JUnit dependency/configuration rather than leaving
conflicting test dependencies.

``` xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
        </plugin>
    </plugins>
</build>
```

Save the file. In Eclipse, right-click the project → **Maven → Update
Project** if dependencies do not refresh.

## 4. Create the Student Grade Calculator

Under `src/main/java`, inside package `com.example`, replace `App.java`
with `GradeCalculator.java`:

``` java
package com.example;

public class GradeCalculator {

    public static int calculateTotal(int m1, int m2, int m3) {
        return m1 + m2 + m3;
    }

    public static double calculateAverage(int m1, int m2, int m3) {
        return calculateTotal(m1, m2, m3) / 3.0;
    }

    public static boolean isPass(double average) {
        return average >= 40.0;
    }

    public static void main(String[] args) {
        int m1 = 75, m2 = 68, m3 = 82;

        int total = calculateTotal(m1, m2, m3);
        double average = calculateAverage(m1, m2, m3);

        System.out.println("Total   : " + total);
        System.out.println("Average : " + average);
        System.out.println("Result : " + (isPass(average) ? "PASS" : "FAIL"));
    }
}
```

Run `GradeCalculator.java` using **Run As → Java Application**. Expected
console values:

``` text
Total   : 225
Average : 75.0
Result : PASS
```

## 5. Create the JUnit 5 test class

Under `src/test/java`, inside package `com.example`, replace the
generated `AppTest.java` with `GradeCalculatorTest.java`:

``` java
package com.example;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

class GradeCalculatorTest {

    @Test
    void testTotal() {
        assertEquals(225, GradeCalculator.calculateTotal(75, 68, 82));
    }

    @Test
    void testAverage() {
        assertEquals(75.0, GradeCalculator.calculateAverage(75, 68, 82));
    }

    @Test
    void testPass() {
        assertTrue(GradeCalculator.isPass(75.0));
    }

    @Test
    void testFail() {
        assertFalse(GradeCalculator.isPass(35.0));
    }
}
```

Right-click `GradeCalculatorTest.java` → **Run As → JUnit Test**. Verify
all four tests pass.

## 6. Run Maven lifecycle goals in Eclipse

For each command below, right-click the project → **Run As → Maven
build...**, enter the goal, and click **Run**.

  Goal                Purpose
  ------------------- --------------------------------------------------
  `clean`             Removes previous `target` build output
  `test`              Compiles and executes JUnit tests
  `clean package`     Cleans, compiles, tests and packages the project
  `dependency:tree`   Displays JUnit and transitive dependencies

For the pipeline, run:

``` powershell
mvn clean package
```

Expected result: `BUILD SUCCESS`.

Refresh the project and inspect:

``` text
target/
├── classes/
├── test-classes/
├── surefire-reports/
└── maven-github-demo-1.0-SNAPSHOT.jar
```

The supplied Experiment 3 creates the JAR but does not configure it as
an executable/fat JAR. The Dockerfile in this README therefore runs the
compiled application class from `target/classes`, rather than assuming
the JAR has a `Main-Class` manifest or bundled dependencies.

## 7. Add `.gitignore`

Create `.gitignore` at the project root:

``` gitignore
target/
.classpath
.project
.settings/
*.log
```

Do not commit Maven's generated `target/` folder.

## 8. Push the Maven project to GitHub

Create a GitHub repository named `maven-github-demo` (or use your own
repository name). Do not initialize it with a README if you already have
a local project.

Open PowerShell in the project root and run:

``` powershell
git init
git add .
git commit -m "Initial Maven project"
git branch -M main
git remote add origin https://github.com/<YOUR-USERNAME>/maven-github-demo.git
git push -u origin main
```

Replace `<YOUR-USERNAME>` and the repository URL with your own. If
`origin` already exists, use
`git remote set-url origin <YOUR-REPOSITORY-URL>` instead of adding it
again.

## 9. Add the Maven GitHub Actions build

Create `.github/workflows/maven.yml`:

``` yaml
name: Maven Build

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Build with Maven
        run: mvn clean package
```

Commit and push:

``` powershell
git add .github/workflows/maven.yml
git commit -m "Add Maven GitHub Actions workflow"
git push origin main
```

Open the GitHub repository → **Actions** → **Maven Build**. Confirm that
checkout, JDK setup and `mvn clean package` succeed. The source
experiment also asks you to demonstrate a failed test/build, inspect the
Actions log, correct the test, and push again.

## 10. Create the Dockerfile for this Maven application

Create a file named exactly `Dockerfile` (no `.txt` extension) in the
project root.

``` dockerfile
# Build stage: Maven compiles and packages the project
FROM maven:3.9.6-eclipse-temurin-17 AS build
WORKDIR /app

COPY pom.xml .
COPY src ./src

RUN mvn clean package -DskipTests

# Runtime stage: run the compiled Java application
FROM eclipse-temurin:17-jre
WORKDIR /app

COPY --from=build /app/target/classes ./classes

CMD ["java", "-cp", "classes", "com.example.GradeCalculator"]
```

Why `-DskipTests` in the image build stage? The Maven/JUnit verification
is already performed before image publishing in the CI workflow below.
The Docker build still compiles and packages the application. For a
local all-in-one validation, run `mvn clean package` first so tests are
executed before building the image.

This container runs the GradeCalculator Java application. It does not
expose a web port because this experiment's Java program is a console
application, not a web server.

## 11. Build and run the Docker image locally

From PowerShell in the project root:

``` powershell
docker build -t maven-github-demo:1.0 .
docker images
docker run --rm maven-github-demo:1.0
```

Expected container output:

``` text
Total   : 225
Average : 75.0
Result : PASS
```

`--rm` removes the stopped container after the program exits. The image
remains on your computer.

## 12. Create a Docker Hub repository

1.  Sign in to Docker Hub.
2.  Open **My Hub** and choose **Create repository**.
3.  Repository name: `maven-github-demo`.
4.  Select Public or Private as required.
5.  Create the repository.

Your image name will follow this format:

``` text
<DOCKERHUB_USERNAME>/maven-github-demo:1.0
```

Use your own Docker Hub username. Do not copy usernames or repository
names from sample screenshots.

## 13. Generate a Docker Hub access token

Do not put your Docker Hub account password in scripts or GitHub
workflow files.

1.  In Docker Hub, open your profile avatar → **Account Settings**.
2.  Open **Security**.
3.  Under **Personal access tokens**, choose **Generate new token**.
4.  Give it a name such as `Maven-GitHub-Publish`.
5.  Grant **Read & Write** permissions for pushing and pulling images.
6.  Generate the token and copy it immediately. Docker Hub may not show
    it again.
7.  Store it privately. Never paste it into your README, source code,
    screenshots, or Git repository.

If a real access token is ever committed or exposed, revoke it in Docker
Hub and generate a replacement.

## 14. Log in to Docker Hub and push the image

In PowerShell, tag the local image using your Docker Hub username:

``` powershell
docker tag maven-github-demo:1.0 <DOCKERHUB_USERNAME>/maven-github-demo:1.0
```

Log in. When prompted, enter your Docker Hub username and paste the
access token as the password:

``` powershell
docker login --username <DOCKERHUB_USERNAME>
```

Then push:

``` powershell
docker push <DOCKERHUB_USERNAME>/maven-github-demo:1.0
```

Check Docker Hub → your repository → **Tags**. The `1.0` tag should
appear.

## 15. Pull and run the image on another computer

On Computer 2:

1.  Install Docker Desktop and make sure it is running.
2.  Open PowerShell.
3.  For a private repository, log in using an account with pull
    permission. Public images can generally be pulled without logging
    in.

``` powershell
docker login
docker pull <DOCKERHUB_USERNAME>/maven-github-demo:1.0
docker images
docker run --rm <DOCKERHUB_USERNAME>/maven-github-demo:1.0
```

Expected output is the same GradeCalculator output. This verifies that
the image stored on Docker Hub can be downloaded and executed on another
computer.

## 16. Optional: make GitHub Actions publish the image automatically

This is the CI extension of the manual Docker Hub process. It keeps the
Maven test/build job separate from the image publishing job. The image
is published only for a push to `main`, and only after the Maven job
succeeds.

### 16.1 Add GitHub repository secrets

In GitHub, open your repository → **Settings → Secrets and variables →
Actions → New repository secret**.

Create:

  Secret name            Value
  ---------------------- ------------------------------------------
  `DOCKERHUB_USERNAME`   Your Docker Hub username
  `DOCKERHUB_TOKEN`      The Docker Hub Read & Write access token

Do not hard-code either value in YAML.

### 16.2 Replace `.github/workflows/maven.yml` with this workflow

``` yaml
name: Maven Build and Docker Hub Publish

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  maven-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Build and test with Maven
        run: mvn clean package

  build-and-publish:
    needs: maven-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/maven-github-demo:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/maven-github-demo:${{ github.sha }}
```

Commit and push:

``` powershell
git add Dockerfile .github/workflows/maven.yml
git commit -m "Add Docker image build and Docker Hub publishing"
git push origin main
```

Open GitHub → **Actions**. The `build-and-publish` job must wait for
`maven-test`. After a successful push to `main`, check Docker Hub for
the `latest` tag and the commit-SHA tag.

## 17. Troubleshooting

  ------------------------------------------------------------------------------------
  Problem                             Check
  ----------------------------------- ------------------------------------------------
  `docker` is not recognized          Install Docker Desktop, restart the terminal and
                                      verify PATH

  Cannot connect to Docker daemon     Start Docker Desktop and wait for Engine to run

  Dockerfile not found                Run commands from the project root; confirm the
                                      file is named `Dockerfile`

  Maven tests fail                    Run `mvn clean test` and inspect the Surefire
                                      reports under `target/surefire-reports/`

  Push denied                         Confirm login, token Read & Write permission,
                                      repository name and username

  Image not found locally             Run `docker images` and verify the exact tag

  Wrong image name                    Use
                                      `<DOCKERHUB_USERNAME>/maven-github-demo:<tag>`
                                      consistently

  Container exits                     This is a short console program; use
                                      `docker run --rm ...` to see its output
  ------------------------------------------------------------------------------------

## 18. Final project structure

``` text
maven-github-demo/
├── .github/
│   └── workflows/
│       └── maven.yml
├── src/
│   ├── main/
│   │   └── java/com/example/GradeCalculator.java
│   └── test/
│       └── java/com/example/GradeCalculatorTest.java
├── Dockerfile
├── .gitignore
└── pom.xml
```

## 19. Complete command sequence

Run from the project root. Replace the Docker Hub placeholder before
executing the tag and push commands.

``` powershell
mvn clean package
docker build -t maven-github-demo:1.0 .
docker run --rm maven-github-demo:1.0

docker tag maven-github-demo:1.0 <DOCKERHUB_USERNAME>/maven-github-demo:1.0
docker login --username <DOCKERHUB_USERNAME>
docker push <DOCKERHUB_USERNAME>/maven-github-demo:1.0

# On another computer
docker pull <DOCKERHUB_USERNAME>/maven-github-demo:1.0
docker run --rm <DOCKERHUB_USERNAME>/maven-github-demo:1.0
```

## 20. Result

The Student Grade Calculator Maven project is created in Eclipse,
configured with JUnit 5, tested and packaged using Maven, stored in
GitHub, and built as a Docker image. The image is tagged and pushed to
Docker Hub, then pulled and run on another computer. The optional GitHub
Actions extension automates Docker image publishing after the Maven
build succeeds.
