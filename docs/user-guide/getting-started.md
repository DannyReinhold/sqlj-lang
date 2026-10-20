<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Getting started

Use this guide to try the installed plugin. The plugin source is private; no
clone, Gradle build or developer token is required. For managed corporate IDEs,
ask your platform team to follow [deployment guidance](../administration/deployment.md).
Read the [compatibility matrix](compatibility.md) before evaluating a new environment.

## 1. Install an approved package

Obtain the plugin ZIP from the maintainer/your administrator. In IntelliJ open
**Settings → Plugins → gear menu → Install Plugin from Disk**, select the ZIP and
restart when requested. Confirm **SQLJ Language Support** is enabled under Installed. For a Marketplace
package, activate its subscription/trial in the IDE's license management.
Tools → SQLJ: License Information shows status and opens activation. A privately
supplied customer package uses its separate written grant instead.
The [Marketplace listing](https://plugins.jetbrains.com/plugin/35279) provides installation and trial/subscription
options once approved and published. In Settings → Plugins → Marketplace, search
for **SQLJ Language Support** and check that the vendor is Danny Reinhold.

Java support is required. Maven support is required for the demo import and its
before-launch task. Optional Database Tools/SQL support adds SQL dialect/schema
assistance; it is not needed for basic file recognition. Obtain the necessary
plugin entitlement before commercial deployment; see [licensing](../licensing.md).

## 2. Prepare the demo environment

Have **JDK 21**, Maven, Docker and Docker Compose v2 available to the build.
The Compose version must support `up --wait --wait-timeout`. Check from a terminal:

```powershell
java -version
mvn -version
docker version
docker compose version
```

Make JDK 21 available to command-line Java/Maven and select it as the project SDK
and Maven Runner JRE. The IDE's own runtime is separate. Initial downloads need
network access. Ensure ports **50000** (DB2) and **8080** (demo HTTP) are available.
The generated container has a fixed name, `db2-sqlj-demo`; avoid parallel demo
projects using that same container/port unless you adapt their configuration.

The Compose template is a **local development example**, with a privileged DB2
container and demo credentials. Review `compose.yaml` under your organization's
container policy. It is not a production deployment template. IBM image/tool
license terms apply independently. The generated startup commands use Docker and Compose. A Podman environment
requires adapting that toolchain; it is not a drop-in option supplied by the wizard.

## 3. Create and import the project

Choose **File → New → Project → SQLJ → SQLJ Spring Boot Application**. Set Maven
coordinates and the base package, then Create. The wizard generates a Spring Boot
3.4.3/Thymeleaf demo, Maven POM, DB2 Compose file, SQLJ repository, database
migrations, web UI and **SQLJ Demo (Maven + SMAP)** run configuration.

Wait for Maven import. If the Maven tool window has no project, enable Maven
support, right-click `pom.xml` and choose **Add as Maven Project**, then reload.
Keep `.java` in `src/main/java` and `.sqlj` in `src/main/sqlj`. The wizard includes
the SQLJ repository; Java integration is automatic, with no special source comment.

DB2 and its SAMPLE database are initialized automatically. Follow container output
with `docker compose logs --follow db2` from the generated project directory.

## 4. Run the demo

In the generated project directory:

```powershell
mvn spring-boot:run
```

Maven's container profile starts DB2, waits for a successful connection/query,
extracts translator tools, translates the repository, compiles Java and installs
source maps before Spring Boot runs. Flyway creates and seeds the demo tables.
After the application reports ready, open [http://localhost:8080](http://localhost:8080).
Java startup prints elapsed-time activity messages every 15 seconds; these are
not database progress percentages. Before Java starts, use the DB2 logs above.

Later starts normally reuse the database. On timeout inspect, without deleting data:

```powershell
docker compose ps
docker compose logs --tail 100 db2
```

If initialization is progressing but needs more time, retry with a larger budget:

```powershell
mvn spring-boot:run -Ddb2.startup.timeout.seconds=1200
```

Stop Spring Boot with Ctrl+C. Container lifetime is separate; check `docker compose ps`.
Use `docker compose stop db2` to stop the demo database without requesting volume
deletion. Do not use volume-removal commands to troubleshoot a slow first start.

### Translator selection

New starters select tools at build time in this order:

1. `-Dsqlj.executable=<path>` or `-Dsqlj.translator.jar=<path>` when explicitly set.
   Executable selection wins if both are set. An invalid explicit path fails.
2. A `sqlj` executable/script on PATH (`.exe`, `.cmd`, `.bat`, `.com` on Windows).
3. `${user.home}/.m2/repository/com/ibm/db2/sqlj/11.5/sqlj-11.5.jar` if installed.
4. The translator extracted from the ready DB2 demo container into `target/sqlj4.zip`.

The `[SQLJ]` build output identifies the selected tool. Maven supplies the
application dependency classpath; a nonstandard Maven repository location does
not require a hardcoded JDBC JAR path. For example, in PowerShell:

```powershell
mvn spring-boot:run "-Dsqlj.executable=C:/IBM SQLJ/bin/sqlj.cmd"
```

Or select a translator JAR instead of a script:

```powershell
mvn spring-boot:run "-Dsqlj.translator.jar=C:/IBM SQLJ/java/sqlj4.zip"
```

The bundled build launcher translates every `.sqlj` file recursively below
`src/main/sqlj` in one invocation and requests source maps for each. It waits
for database health before container extraction, and stops the build if a tool
or translation step fails. Old `sqlj-local`/`sqlj-container` profiles are not used
by newly generated projects. After removing or renaming SQLJ inputs, run
`mvn clean process-classes` to remove stale generated Java/classes before debugging.
Existing projects keep their current POM until
explicitly updated; plugin installation never rewrites it.

## 5. Try navigation and debugging

Open `TodoRepository.sqlj`. Ctrl+B on a host variable navigates to its Java
declaration; Ctrl+B on `TodoRepository` in `TodoController.java` should
navigate back to SQLJ, including after Maven generates Java in its standard output.
Completion and highlighting depend on completed Maven import and the shared Java view.

For an IDE launch, select **SQLJ Demo (Maven + SMAP)**, then Run or Debug. It runs
`clean process-classes` before launch and blocks launch on build failure. The
ordinary IDE Build task is disabled in this configuration so it cannot overwrite
source maps afterward. Other run configurations are unchanged. If the Maven
module was renamed, select its actual module in Edit Configurations.

Set a breakpoint on an executable Java statement in the SQLJ repository or the
opening `#sql` line, then open the task page. `mvn compile` alone stops too early
to install SMAP. Use the [debugging guide](debugging-project-types.md) for source-map
setup, stepping, expression evaluation and diagnostics.

## 6. Move to your own project

Follow [existing-project editor setup](ide-features.md). Keep your approved
translator/runtime/binding pipeline. New demo projects automatically translate
all SQLJ sources under their configured root. Existing enterprise projects keep
their own translation and binding steps; the plugin does not rewrite them.
For a runnable orientation, continue with the [wizard walkthrough](../tutorials/first-sqlj-program.md).

## The Small Steps demo

The wizard creates a ToDo list with local styling and five starter tasks. Add a task
with an optional note, mark it done or reopen it, delete it, and switch between All,
To do and Done. SQLJ Plugin and Vernac tasks include links to their websites.
The SQLJ repository reads and writes DB2; tasks remain in the named `db2-data`
Compose volume across restarts and container recreation. The ToDo demo uses its own
`sqlj_todo_schema_history` migration table, separate from earlier catalog demos.
The fixed container name and port mean only one demo can run at a time.

Generate the new demo in a fresh project directory. Existing product-catalog demos
keep their original schema and Flyway history; the wizard does not upgrade those projects.
The demo is intended for local development and has no login or user separation.

The plugin supplies a small MIT-licensed build tool at `.mvn/sqlj/sqlj-debug-tool.jar`.
It embeds SQLJ debug metadata during Maven's build, without adding a helper Java class
to your project or a runtime dependency to your application. Commit this build tool
with the project to retain standalone Maven builds. Its version is fixed when the
project is generated; a plugin update does not replace files in existing projects.
