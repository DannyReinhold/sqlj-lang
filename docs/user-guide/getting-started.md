<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Getting started

Use this guide to try the installed plugin. The plugin source is private; no
clone, Gradle build or developer token is required. For managed corporate IDEs,
ask your platform team to follow [deployment guidance](../administration/deployment.md).
Read the [compatibility matrix](compatibility.md) before evaluating a new environment.

## 1. Install an approved package

Obtain the plugin ZIP from the maintainer/your administrator. In IntelliJ open
**Settings → Plugins → gear menu → Install Plugin from Disk**, select the ZIP and
restart when requested. Confirm **SQLJ Language Support** is enabled under Installed.
Once a Marketplace version is published, the listing offers an alternative;
this pre-release guide does not imply that it is already available.

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
license terms apply independently. Docker-based startup is the validated baseline;
rootless Podman equivalence has not been established.

## 3. Create and import the project

Choose **File → New → Project → SQLJ → SQLJ Spring Boot Application**. Set Maven
coordinates and the base package, then Create. The wizard generates a Spring Boot
3.4.3/Thymeleaf demo, Maven POM, DB2 Compose file, SQLJ repository, database
migrations, web UI and **SQLJ Demo (Maven + SMAP)** run configuration.

Wait for Maven import. If the Maven tool window has no project, enable Maven
support, right-click `pom.xml` and choose **Add as Maven Project**, then reload.
Keep `.java` in `src/main/java` and `.sqlj` in `src/main/sqlj`. The wizard includes
the enhanced Java-view marker in its SQLJ repository.

> **FIRST DB2 START: ALLOW MANY MINUTES — PLEASE DO NOT ABORT EARLY.**
> The starter allows **15 minutes for readiness after container startup**;
> image download/build time is additional. The complete first launch can exceed
> 15 minutes. A running container or open port is not proof that SAMPLE is ready.
> In a second terminal, from the generated project directory, use
> `docker compose logs --follow db2` to watch initialization.

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

### Translator selection: current behavior

The generated POM selects `sqlj-local` if
`${user.home}/.m2/repository/com/ibm/db2/sqlj/11.5/sqlj-11.5.jar` exists. Otherwise
it selects `sqlj-container` and extracts tools from the DB2 image into `target`.
**It does not currently search PATH for a `sqlj` executable.** The local profile
also assumes its configured JDBC jar path; using it requires those local artifacts.
Use `mvn help:active-profiles` to inspect the selection. Do not confuse these Maven
profiles with the plugin's separate Toolchain settings; those settings do not
rewrite the generated POM. The container profile is the first tested baseline.

## 5. Try navigation and debugging

Open `ProductRepository.sqlj`. Ctrl+B on a host variable navigates to its Java
declaration; Ctrl+B on `ProductRepository` in `ProductController.java` should
navigate back to SQLJ, including after Maven generates Java in its standard output.
Completion and highlighting depend on completed Maven import and the shared Java view.

For an IDE launch, select **SQLJ Demo (Maven + SMAP)**, then Run or Debug. It runs
`clean process-classes` before launch and blocks launch on build failure. The
ordinary IDE Build task is disabled in this configuration so it cannot overwrite
source maps afterward. Other run configurations are unchanged. If the Maven
module was renamed, select its actual module in Edit Configurations.

Set a breakpoint on an executable Java statement in the SQLJ repository or the
opening `#sql` line, then open the product page. `mvn compile` alone stops too early
to install SMAP. Full stepping/evaluation acceptance is still in progress; use the
[debugging guide](debugging-project-types.md) for setup, limits and diagnostics.

## 6. Move to your own project

Follow [existing-project editor setup](ide-features.md). Keep your approved
translator/runtime/binding pipeline. The demo POM currently translates its one
`ProductRepository.sqlj` input; adding files does not automatically add translation
executions. The plugin does not supply a complete generic Maven/Gradle/Ant build.
For a runnable orientation, continue with the [wizard walkthrough](../tutorials/first-sqlj-program.md).
