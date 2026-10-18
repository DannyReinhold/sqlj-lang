<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# SQLJ Language Documentation Map

Welcome to the documentation for **SQLJ Language Support for IntelliJ IDEA** (`sqlj-lang`) and the SQLJ language tooling ecosystem at **[sqlj-lang.org](https://sqlj-lang.org)**.

---

## Structure Overview

### 1. [Home / Overview](index.md)
The main entry point for the SQLJ language project, features, quick start examples, and architecture overview.

### 2. User Documentation & Reference
- **[User Guide Overview](user-guide/index.md)**: High-level overview of using SQLJ in modern Java environments.
- **[Getting Started & Installation](user-guide/getting-started.md)**: Installing the IntelliJ plugin, configuring JDK 21+ and DB2 dependencies.
- **[SQLJ Language Reference](user-guide/language-reference.md)**: Syntax, clauses (`#sql`, `iterator`, `context`), bind expressions (`:var`, `:OUT`, `:INOUT`), and type mappings.
- **[IDE Features & Code Insight](user-guide/ide-features.md)**: Syntax highlighting, DB2 SQL injection, code completion, navigation, and debugging.

### 3. Tutorials
- **[Tutorials Overview](tutorials/index.md)**: Step-by-step learning paths.
- **[Your First SQLJ Program](tutorials/first-sqlj-program.md)**: Create, write, and execute your first `.sqlj` application.
- **[Working with SQLJ Iterators](tutorials/working-with-iterators.md)**: Named vs. positional iterators, typed cursor traversal, and error handling.
- **[DB2 Connections & Execution Contexts](tutorials/db2-connection-and-contexts.md)**: Multi-connection architectures, custom execution contexts, and transaction boundaries.

### 4. Developer & Contributor Documentation
- **[Developer Portal](development/index.md)**: Contributing, architecture, AST & PSI design, and development principles.
- **[Development Plan](development/development-plan.md)**: Milestone roadmap from Phase 0 to Phase 6.
- **[Language Semantics & Java Integration](development/language-and-java.md)**: PSI models, multi-host injection, scoping, and lessons learned.
- **[Debugging & Verification](development/debugging-and-verification.md)**: Testing strategy, position mapping, and JDI debugger bridge.
- **[SQLJ Design Checklist](development/sqlj-design-checklist.md)**: Practical guidelines for compiler bridges and editor recovery.

### 5. Maintainer & Implementation Guides
- **[Maintainer Guide Index](maintainers/index.md)**: Main hub for maintainers and active contributors.
- **[Building, Running & Local Installation](maintainers/building-and-running.md)**: Detailed Gradle commands (`buildPlugin`, `runIde`, `test`), installing `.zip` archives from disk, and sandbox workflows.
- **[Source Code & Architecture Walkthrough](maintainers/source-code-walkthrough.md)**: Tour through all packages, AST/PSI models, SQL injection, synthetic classes, host variable binding, debugger position manager, and compiler integration.
- **[Releasing & Deployment](maintainers/releasing-and-deployment.md)**: Release checklists, versioning, signing, and JetBrains Marketplace publishing.

### 6. Legal & Policies
- **[Licensing](licensing.md)**: Plugin, documentation and code-example license scopes.
- **[Legal Notice / Impressum](impressum.md)**: Provider and contact details pursuant to DDG §5.
- **[Privacy Notice](privacy.md)**: Data protection and privacy notice for sqlj-lang.org.
