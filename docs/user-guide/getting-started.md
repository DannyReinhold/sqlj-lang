<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Getting Started with SQLJ

This guide walks you through setting up your environment for SQLJ development with IntelliJ IDEA and IBM DB2.

---

## Prerequisites

1. **IntelliJ IDEA**: 2026.1+ (Build `#261.24374-151` or later).
2. **Java Development Kit (JDK)**: JDK 21 LTS or higher.
3. **Database**: IBM DB2 (LUW or z/OS), Apache Derby, or Oracle with SQLJ support.
4. **SQLJ Runtime Libraries**: `db2jcc.jar`, `db2jcc_license_cu.jar` and `sqlj.zip` / `sqlj-runtime.jar`.

---

## 1. Installing the IntelliJ Plugin

1. Open **Settings / Preferences** (`Ctrl+Alt+S` on Windows/Linux, `Cmd+,` on macOS).
2. Navigate to **Plugins** -> **Marketplace**.
3. Search for **`SQLJ Language Support`** (`org.sqlj.lang`).
4. Click **Install** and restart the IDE if prompted.

Alternatively, for offline or SNAPSHOT installations:
- Go to **Plugins** -> ⚙ (Gear icon) -> **Install Plugin from Disk...**.
- Select the packaged `sqlj-lang-0.1.0-SNAPSHOT.zip`.

---

## 2. Creating a Project via the "New Project" Wizard

The plugin provides a dedicated **New Project Wizard** to instantly spin up a complete Spring Boot + Thymeleaf demo application with a local DB2 Docker container and sample `.sqlj` files:

1. Click **File** -> **New** -> **Project...** (or select **New Project** on the Welcome Screen).
2. Select **SQLJ Spring Boot Application** under the **SQLJ** group.
3. Configure your Maven coordinates (`GroupId`, `ArtifactId`, `Version`, and base package name).
4. Click **Create**.

IntelliJ IDEA generates:
- A modern Spring Boot 3.4 application with Thymeleaf and Spring JDBC.
- Full Maven build integration with dual-strategy SQLJ translation (local `sqlj-local` profile when `com.ibm.db2:sqlj` exists in `~/.m2`, or automated `sqlj-container` profile that starts the container, extracts `sqlj4.zip`, and translates SQLJ sources).
- Automatic Docker Compose integration via `spring-boot-docker-compose`, launching local IBM DB2 from `compose.yaml` automatically on application startup.
- Automatic database migrations and data seeding via **Flyway** (`src/main/resources/db/migration/V1__init_schema.sql` and `V2__sample_data.sql`).
- A working `ProductRepository.sqlj` using typed SQLJ iterators, `DefaultContext`, and host variables (`:targetCategory`, `:newStock`).
- A Thymeleaf UI with a product catalog and interactive stock updates.

To run the generated project:
```bash
# Start the Spring Boot application (DB2 container starts automatically via Docker Compose integration!)
mvn spring-boot:run
```
Then open [http://localhost:8080](http://localhost:8080) in your browser.

---

## 3. Manual Project Setup & Build Configuration

SQLJ source files use the `.sqlj` extension. To translate and compile them, you configure your build tool (Maven, Gradle, or Ant) to execute the SQLJ translator.

### Maven Configuration

Add the SQLJ runtime dependency to your `pom.xml`:

```xml
<dependencies>
    <!-- DB2 JDBC Driver and SQLJ Runtime -->
    <dependency>
        <groupId>com.ibm.db2</groupId>
        <artifactId>jcc</artifactId>
        <version>11.5.8.0</version>
    </dependency>
</dependencies>
```

Configure the SQLJ translation goal (or use the `exec-maven-plugin` / `sqlj-maven-plugin`):

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>exec-maven-plugin</artifactId>
            <version>3.1.0</version>
            <executions>
                <execution>
                    <id>sqlj-translate</id>
                    <phase>generate-sources</phase>
                    <goals>
                        <goal>exec</goal>
                    </goals>
                    <configuration>
                        <executable>sqlj</executable>
                        <arguments>
                            <argument>-dir=target/generated-sources/sqlj</argument>
                            <argument>-d=target/classes</argument>
                            <argument>src/main/sqlj/**/*.sqlj</argument>
                        </arguments>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Gradle Configuration (`build.gradle.kts`)

```kotlin
plugins {
    java
}

dependencies {
    implementation("com.ibm.db2:jcc:11.5.8.0")
    // Include SQLJ runtime jar in your compile classpath
    compileOnly(files("libs/sqlj.zip"))
}

sourceSets {
    main {
        java {
            srcDir("src/main/sqlj")
            srcDir("build/generated/sources/sqlj")
        }
    }
}
```

---

## 4. Creating Your First `.sqlj` File

1. Right-click on your source folder (`src/main/java` or `src/main/sqlj`).
2. Select **New** -> **SQLJ File**.
3. Name your file (e.g. `CustomerRepository.sqlj`).
4. IntelliJ automatically assigns the SQLJ icon and provides syntax highlighting and error checking.

---

## 5. Initializing the Default Connection Context

In SQLJ, queries execute against a connection context. You can set the default context using standard JDBC:

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import sqlj.runtime.ref.DefaultContext;

public class App {
    public static void main(String[] args) throws SQLException {
        Connection conn = DriverManager.getConnection(
            "jdbc:db2://localhost:50000/SAMPLE", "db2inst1", "password"
        );
        DefaultContext.setDefaultContext(new DefaultContext(conn));

        // Now all #sql statements using default context will execute on 'conn'
    }
}
```

---

## 6. Working with Local DB2 (Docker & Testcontainers)

For local development without an installed enterprise DB2 server:

### Running DB2 in Docker / Podman
```bash
docker run -d --name db2 -p 50000:50000 \
  -e DB2INST1_PASSWORD=password -e LICENSE=accept \
  icr.io/db2_community/db2:11.5.9.0
```
In IntelliJ IDEA, select **Settings** -> **Languages & Frameworks** -> **SQLJ & DB2 Toolchain**, choose **Docker Container**, and verify the connection.

### Integration Testing with Testcontainers
```java
@Testcontainers
class EmployeeDaoTest {
    @Container
    static Db2Container db2 = new Db2Container("icr.io/db2_community/db2:11.5.9.0")
            .acceptLicense();

    @BeforeAll
    static void setup() throws SQLException {
        Connection conn = DriverManager.getConnection(
            db2.getJdbcUrl(), db2.getUsername(), db2.getPassword()
        );
        DefaultContext.setDefaultContext(new DefaultContext(conn));
    }
}
```

Next: Explore the [SQLJ Language Reference](language-reference.md) or follow [Your First SQLJ Program](../tutorials/first-sqlj-program.md).
