<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# IntelliJ IDE Features & Code Insight

The `sqlj-lang` plugin equips IntelliJ IDEA with rich, context-aware editing and debugging capabilities for SQLJ files.

---

## 1. Syntax Highlighting & Color Schemes

The plugin provides high-fidelity syntax highlighting for hybrid Java and SQLJ files:

- **SQLJ Directives & Keywords**: `#sql`, `iterator`, `context`, `with` highlighted in distinctive colors.
- **Host Variables & Bind Modes**: `:varName`, `:IN`, `:OUT`, `:INOUT` highlighted with special bind-variable formatting.
- **Color Settings Page**: Customize colors via **Settings** -> **Editor** -> **Color Scheme** -> **SQLJ**.

<p align="center">
  <img src="../assets/web/editor-illustration.svg" alt="SQLJ Highlighting and Editor Features" width="750" style="max-width: 100%; height: auto;">
</p>

---

## 2. Multi-Host DB2 SQL Injection

Inside any `#sql { ... }` execution block, IntelliJ injects the database language dialect (e.g. IBM DB2 SQL):

- **Keyword Highlighting & Formatting**: SQL keywords (`SELECT`, `FROM`, `WHERE`, `JOIN`, `CURRENT TIMESTAMP`) are colored and formatted according to your SQL style rules.
- **SQL Code Inspections**: Dialect-specific warnings and inspections flag SQL syntax mistakes without breaking the surrounding Java parser tree.
- **Database Schema Completion**: When connected to an active Database Tool window DataSource, table and column names auto-complete dynamically.

---

## 3. Bidirectional Navigation & Code Insight

### Navigation (`Go to Declaration` / `Find Usages`)
- **Host Variables**: Press `Ctrl+B` (Windows/Linux) or `Cmd+B` (macOS) on `:myVariable` inside an SQL block to jump directly to the Java field, parameter, or local variable declaration.
- **Java References in SQLJ**: Jump directly to declarations of Java classes, methods, parameters, fields, and variables used in regular Java code within `.sqlj` files.
- **Iterators & Contexts**: Jump directly to `#sql iterator MyIter(...)` and `#sql context MyCtx` declarations from any usage point across Java and SQLJ files.
- **Project Java Classes**: Classes defined in `.sqlj` files (e.g. `ProductRepository`) are exposed to the IDE's Java type system via synthetic class providers, enabling seamless navigation from other Java sources.

### Code Completion
- **Java & SQLJ Keywords**: Intelligent completion for standard Java keywords (`class`, `return`, `try`, `finally`, etc.) as well as SQLJ directives (`#sql`, `iterator`, `context`).
- **Scope Variables & Fields**: Suggestions for all visible local variables, parameters, and fields within the active method or class scope.
- **Member & Iterator Methods**: Typing `myIter.` in Java code auto-completes generated iterator column accessors (`id()`, `name()`, `price()`) and control methods (`next()`, `close()`).
- **SQL & Host Binds**: Inside `#sql` blocks, suggestions cover DB2 SQL keywords, bind modes (`IN`, `OUT`, `INOUT`), and host variables (`:varName`).

### Refactoring & Rename
- **Host Variables**: Renaming a Java variable or parameter automatically renames all corresponding `:hostVar` references in embedded `#sql` blocks and vice versa.

---

## 4. Source-Level Debugging

You can debug `.sqlj` files directly without needing to inspect intermediate generated `.java` files:

1. **Line Breakpoints**: Click on the editor gutter on any executable Java or `#sql` line.
2. **Debugger Stepping**: Use `Step Over` (`F8`) and `Step Into` (`F7`) to navigate between Java and SQL statements.
3. **Expression Evaluation & Watches**: Evaluate expressions (`Alt+F8`) within the live lexical context of the stack frame.
4. **Position Mapping**: The debugger bridge accurately maps JVM bytecode execution points back to original `.sqlj` line offsets.
5. **Stepping Filter**: Automatically skips over internal SQLJ runtime classes (`sqlj.runtime.*`, `com.ibm.db2.jcc.sqlj.*`) when stepping through code.

---

## 5. DB2 Toolchain & Environment Settings

Configure the SQLJ translation pipeline and target database environment in **Settings** -> **Languages & Frameworks** -> **SQLJ & DB2 Toolchain**:

- **Target DB2 Environments**:
  - **Installed DB2 Instance**: Connects to enterprise or local DB2 instances (LUW / z/OS).
  - **Docker / Podman Container**: Optimised default JDBC URL (`jdbc:db2://localhost:50000/SAMPLE`) and credential mappings for containerised databases.
  - **Testcontainers**: Seamless support for ephemeral test containers with automated connection parameters.
- **Online Semantics Checking**: Enables compile-time table and column validation against the live DB2 database catalog (`-url`, `-user`, `-password`, `-default-schema`).
- **Compiler Runner & Diagnostics**: Customisable translator executable path (`sqlj` / `db2sqljcompiler`) with inline message mapping to `.sqlj` source lines.
- **Generated Sources Management**: Automatically manages and registers output directories (`build/generated/sources/sqlj` or `target/generated-sources/sqlj`) as project source roots.

---

## 6. "New Project" Wizard (Spring Boot & DB2 Starter)

The plugin includes an integrated Project Wizard for IntelliJ IDEA:

- **Template Generation**: Creates a runnable Spring Boot 3.4 + Thymeleaf application configured with Spring JDBC and IBM DB2 driver (`jcc`).
- **One-Command Local DB2 Container**: Generates a pre-configured `compose.yaml` spinning up a local DB2 instance with a `SAMPLE` database and healthchecks.
- **Embedded SQLJ Repository & Web UI**: Includes a working SQLJ repository with typed iterators and bind variables (`ProductRepository.sqlj`) and an interactive Thymeleaf UI for browsing and updating records.
