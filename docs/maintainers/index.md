<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# SQLJ Plugin Maintainer & Contributor Guide

Welcome to the internal maintainer and core developer documentation for the **SQLJ Language Support Plugin (`sqlj-lang`)**.

This section serves as a practical, high-efficiency handbook for developers working on the plugin itself. It explains how to build, test, and debug the plugin locally, and provides an in-depth architectural walkthrough of the entire codebase.

---

## Maintainer Topics

### 1. [Building, Running & Local Installation](building-and-running.md)
Step-by-step instructions for:
- Building distribution archives (`buildPlugin`)
- Performing clean builds and troubleshooting cache issues
- Installing the plugin manually from disk in IntelliJ IDEA
- Launching the sandboxed development IDE (`runIde`)
- Running unit and integration test suites (`test`)
- Verifying platform compatibility with IntelliJ Plugin Verifier (`runPluginVerifier`)

### 2. [Source Code & Architecture Walkthrough](source-code-walkthrough.md)
Comprehensive tour through all packages and architectural subsystems:
- **Lexer, Parser & PSI**: How `.sqlj` files are tokenized, parsed, and represented in IntelliJ's Program Structure Interface (PSI).
- **Multi-Host Language Injection**: Injecting DB2 SQL dialect into `#sql` blocks.
- **Bidirectional Host Variable Binding**: Resolving `:variable` bindings to Java declarations (parameters, local variables, fields).
- **Synthetic PSI Classes**: How `SqljSyntheticIteratorClass` and `SqljSyntheticContextClass` expose SQLJ iterators/contexts to IntelliJ's Java type system.
- **Code Insight, Highlighting & Navigation**: Annotators, syntax highlighters, completion contributors, and Find Usages.
- **Source-Level Debugger Bridge**: How `SqljPositionManager` maps breakpoints, stack frames, and expression evaluation between `.sqlj` sources and translated `.java` bytecode.
- **Toolchain & Compiler Runner**: External SQLJ translator integration (`db2sqljcompiler` / `sqlj`) and diagnostic error parsing.
- **Project Wizard & Template Generator**: How `SqljModuleBuilder` and `SqljProjectTemplateGenerator` create ready-to-use Spring Boot + DB2 Docker projects.

### 3. [Releasing & Publishing](releasing-and-deployment.md)
Guidance on versioning, signing, plugin verification, and publishing to the JetBrains Marketplace.

---

## Quick Reference Commands

| Task | Command (Linux / macOS) | Command (Windows PowerShell) | Output / Action |
|---|---|---|---|
| **Run all tests** | `./gradlew test` | `.\gradlew.bat test` | Executes JUnit / IntelliJ test fixtures |
| **Build installable ZIP** | `./gradlew buildPlugin` | `.\gradlew.bat buildPlugin` | Creates ZIP in `build/distributions/` |
| **Clean & Build ZIP** | `./gradlew clean buildPlugin` | `.\gradlew.bat clean buildPlugin` | Fresh build avoiding cached artifacts |
| **Launch Sandbox IDE** | `./gradlew runIde` | `.\gradlew.bat runIde` | Spawns isolated IntelliJ test instance |
| **Plugin Verifier** | `./gradlew runPluginVerifier` | `.\gradlew.bat runPluginVerifier` | Checks binary compatibility |

---

## Related Documentation
- [Root Contributing Guide](../../CONTRIBUTING.md)
- [Development Roadmap & Plan](../development/development-plan.md)
- [Design Checklist & Lessons Learned](../development/sqlj-design-checklist.md)
- [Language & Java Architecture](../development/language-and-java.md)
- [Debugging & Verification Architecture](../development/debugging-and-verification.md)
- [Full Documentation Map](../documentation-map.md)
