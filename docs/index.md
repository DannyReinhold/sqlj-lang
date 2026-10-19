<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

<p align="center">
  <img src="assets/web/social-preview.svg" alt="SQLJ Language Support" width="600">
</p>

# SQLJ Language Support for IntelliJ IDEA

Edit Java with embedded SQL in `.sqlj` files: host-variable completion,
Java/SQLJ navigation and DB2-oriented SQL assistance. A project wizard creates a
Spring Boot demonstration with a local DB2 LUW container.

**Current status: pre-release, under stabilization.** Source breakpoints and Maven
preparation have been confirmed in the DB2 wizard application. Stepping, watches,
conditions and logpoints are implemented but still awaiting full installed-IDE
acceptance. Oracle SQLJ, Derby and other DB2 server/toolchain combinations are
not supported release claims. See the [capability matrix](user-guide/compatibility.md).

## Choose your starting point

| Your responsibility | Start here |
| --- | --- |
| Evaluate, approve or distribute the plugin | [Administration and deployment](administration/deployment.md) |
| Install and try the DB2 demo | [Getting started](user-guide/getting-started.md) |
| Work on an existing SQLJ codebase | [Editor features and project setup](user-guide/ide-features.md) |
| Debug SQLJ code | [Debugging setup](user-guide/debugging-project-types.md) |
| Diagnose a problem | [Troubleshooting and support](user-guide/troubleshooting.md) |
| Check entitlement and reuse of examples | [Licensing](licensing.md) |

> **FIRST DB2 START: ALLOW MANY MINUTES.** The image download and SAMPLE database
> creation can be slow. The starter waits up to **15 minutes for readiness after
> container startup**, in addition to download/build time. The complete cold launch
> can exceed 15 minutes. Keep the build running and follow
> `docker compose logs --follow db2` in another terminal. A running container does
> not yet mean that the database is ready.

## What the plugin and the translator do

The plugin supplies IDE assistance. The IBM SQLJ translator, JDBC/SQLJ runtime,
database privileges and any required customization/binding remain part of your
application's build and deployment. SQL/schema checking depends on the actual
translator configuration or the IDE's optional database tooling; this plugin does
not independently validate every statement against a live database.

For example, in a configured SQLJ class with an available connection context:

```java
int employeeId = 42;
String name = null;
#sql { SELECT name INTO :name FROM employee WHERE id = :employeeId };
```

`:employeeId` reads a Java value; `:name` is a write target in `SELECT INTO`.
Use Ctrl+B to navigate to their declarations. This fragment assumes the table,
connection and application build already exist. The [wizard walkthrough](tutorials/first-sqlj-program.md)
is the runnable starting point.

## Distribution and ownership

The source repository is private. The [GitHub repository](https://github.com/DannyReinhold/sqlj-lang)
contains generated public documentation and website assets only; it cannot be
cloned to build the plugin. Obtain an approved plugin ZIP from the maintainer,
or use its Marketplace listing once published. No Marketplace release is implied
by this pre-release documentation.

This is an independent plugin by Danny Reinhold, not an official IBM or JetBrains
product. See [licensing](licensing.md), [legal notice](impressum.md) and
[privacy](privacy.md). [All documentation](documentation-map.md).
