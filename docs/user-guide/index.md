<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# SQLJ User Guide

Welcome to the SQLJ User Guide. This guide covers how to set up, write, and execute SQLJ programs in modern Java and IntelliJ IDEA environments.

---

## Guide Contents

1. **[Getting Started & Installation](getting-started.md)**  
   How to install the plugin, configure runtime dependencies (`sqlj.runtime`), setup DB2 JDBC drivers, and configure your build system.

2. **[SQLJ Language Reference](language-reference.md)**  
   Complete specification of SQLJ syntax:
   - `#sql` statements and execution blocks
   - Named and positional iterators (`#sql iterator ...`)
   - Connection context declarations (`#sql context ...`)
   - Host variable bindings (`:var`, `:IN`, `:OUT`, `:INOUT`, `:(expr)`)
   - Transaction control and context qualifiers (`#sql [myCtx] { ... }`)

3. **[IDE Features & Code Insight](ide-features.md)**  
   Productivity features in IntelliJ IDEA:
   - Dual-language syntax highlighting and color settings
   - Embedded DB2 SQL code completion (tables, columns, dialect syntax)
   - Host variable reference navigation (`Ctrl+B`)
   - Source-level breakpoints; stepping and expression evaluation remain under verification

4. **[Debugging Java, Spring & WebSphere Liberty](debugging-project-types.md)**

   IDE and Maven setup for main-class applications and application servers,
   source-map preservation, packaging/deployment, remote attach and acceptance status.

---

## Architectural Principles of SQLJ

SQLJ separates relational database interactions from procedural Java logic while maintaining strict compile-time type safety:

- **Static SQL vs. Dynamic JDBC**: Static SQL queries are known at compile time, enabling static database package creation, database access authorization checks, and optimized access paths.
- **Embedded Syntax**: SQL statements are written naturally within Java classes, enclosed in `#sql { ... }` blocks.
- **Strong Typing**: Query results are mapped directly to typed iterator methods or variables, reducing boilerplate object-relational mapping code.
