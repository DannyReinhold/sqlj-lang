<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Capabilities, compatibility and current limits

This page describes the plugin's features, requirements and integration boundaries.
For managed installations, use the approved plugin/IDE combination from your
organization's software distribution policy and the delivered release notes.

## Environment

| Area | Current scope |
| --- | --- |
| IDE | The package declares minimum IntelliJ IDEA 2026.1.2 build `261.24374.151`. Use a compatible IntelliJ IDEA installation with Java support. Other JetBrains IDEs are not covered by this guide. |
| Java support | Requires the IDE's Java plugin. Use the IDE's supported bundled runtime; do not replace it to satisfy application requirements. |
| Demo build | JDK 21, Maven, Docker and Compose v2 with `up --wait --wait-timeout`; network access for initial dependencies/image download. |
| SQL assistance | DB2 language injection when available, otherwise SQL/GenericSQL fallback. Rich schema assistance requires optional Database Tools/SQL support, its entitlement and a configured data source. |
| DB2 baseline | Generated wizard project, DB2 LUW container image `icr.io/db2_community/db2:11.5.9.0`. |
| Oracle SQLJ and Derby | Not currently supported. |
| Build systems | Generated Maven starter and Maven before-launch preparation. Existing builds need their own translation/runtime setup. No automatic Gradle/Ant debug-build adapter. |

The plugin does not bundle an IBM translator, database or licensed driver suite.
Follow the terms for each separately obtained component.

## Editor and SQLJ forms

Java completion, error highlighting and navigation require the
[existing-project setup](ide-features.md#prepare-an-existing-project), including
source roots and runtime dependencies. Java integration is automatic; a source
comment is not required. These editor features
work before SQLJ translation; building and running still use your own toolchain.

| Feature | Implemented behavior and boundary |
| --- | --- |
| Java inside SQLJ | Java declaration/type/member insight through a shared Java view; missing imports/types and invalid calls are highlighted. Translation is not required for this editor model. |
| Java → SQLJ | Navigation to source classes/methods and modeled iterator/context declarations, including nested declarations. Generated Java in the standard Maven output is redirected to SQLJ where matched. |
| Named iterators | Declared column accessors and modeled runtime API available to completion/navigation. |
| Positional iterators | Type-only declarations, positional runtime API and recognition of `FETCH` forms. Query/result compatibility remains the translator's responsibility. |
| Binds | `IN`, `OUT`, `INOUT`, names and parenthesized Java expressions; Java lexical scope, shadowing and member resolution. Bare binds also act as output targets where SQLJ syntax requires it. |
| Write targets | Output/INOUT target checks include writable variables, fields and array elements; a computed method result is not a writable target. |
| Indicators | Primitive `short` checks; variables, fields and array elements (including Java index expressions), parenthesized forms and input expressions. Output indicators must also be writable. |
| Header expressions | Context qualifiers and iterator assignment targets use Java expression insight. ConnectionContext/ExecutionContext roles are checked when runtime types are resolvable. |
| Declaration options | Modeled iterator/context options and context constructor families, including `dataSource`; runtime/JNDI configuration is still the application's responsibility. |
| SQL validation | SQL injection delegates to available IDE SQL tooling. Complete DB2 grammar, schema matching and translator semantics are not independently implemented. |
| Rename / usages | References and rename handlers exist for selected host variables. Cross-language coverage is limited to modeled references; inspect previews/diffs. |

Checks can be incomplete while typing or when runtime classes cannot be resolved.
A file without red highlighting is not proof that the translator or database will
accept it. Use the [language reference](language-reference.md) for supported forms and examples.

## Debugging

| Capability | Behavior / requirements |
| --- | --- |
| Breakpoints in original SQLJ | Stop on executable SQLJ source lines using SMAP embedded in loaded classes. |
| Maven preparation before launch | Runs the configured imported Maven build and prevents launch on failure. Customer POMs are not modified automatically. |
| Stepping and locals | Step Over, Step Into and Step Out with SQLJ source coordinates; inspect parameters and initialized locals in the selected frame. Runtime-package filters reduce SQLJ/DB2 implementation stops. |
| Watches and Evaluate Expression | Java expressions use the selected SQLJ frame's lexical context. |
| Conditions and logpoints | Java boolean conditions, evaluate-and-log expressions and native suspend policies on SQLJ line breakpoints. |
| Session diagnostics | Inspect loaded maps, source fingerprints, class loaders and context metadata. Variable values are not collected in the report. |
| Java/Spring and Liberty | Java debugger integration independent of Spring Boot. Main-class launch and server attach both require matching mapped classes and source. Server deployment remains part of your build/operations workflow. |
| HotSwap and dev-mode rebuilds | No automatic SQLJ translation/map restoration after arbitrary recompilation. Rebuild matching metadata and restart the debug session when output changes. |

See [debugging setup](debugging-project-types.md) and
[step-by-step debugger examples](debugging-checks.md).
