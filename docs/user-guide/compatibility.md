<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Capabilities, compatibility and current limits

Status reviewed **18 October 2026**, during pre-release stabilization. This page
separates functionality present in the plugin from installed-IDE acceptance.
Before production rollout, administrators should validate the exact plugin/IDE
combination described in the delivered release notes.

## Environment

| Area | Current scope |
| --- | --- |
| IDE | The package declares minimum IntelliJ build `261.24374`. Installed-plugin checks have used IDEA Ultimate `262.10968.63`. This is not certification of every newer build or other JetBrains IDE. |
| Java support | Requires the IDE's Java plugin. Use the IDE's supported bundled runtime; do not replace it to satisfy application requirements. |
| Demo build | JDK 21, Maven, Docker and Compose v2 with `up --wait --wait-timeout`; network access for initial dependencies/image download. |
| SQL assistance | DB2 language injection when available, otherwise SQL/GenericSQL fallback. Rich schema assistance requires optional Database Tools/SQL support, its entitlement and a configured data source. |
| DB2 baseline | Generated wizard project, DB2 LUW container image `icr.io/db2_community/db2:11.5.9.0`. |
| z/OS and IBM i | Not yet validated as server/toolchain targets. Similar SQL syntax does not establish binding, deployment or debugging compatibility. |
| Oracle SQLJ and Derby | Not supported/validated in this release scope. Oracle integration is deferred. |
| Build systems | Generated Maven starter and Maven before-launch preparation. Existing builds need their own translation/runtime setup. No automatic Gradle/Ant debug-build adapter. |

The plugin does not bundle an IBM translator, database or licensed driver suite.
Follow the terms for each separately obtained component.

## Editor and SQLJ forms

Most Java integration below requires the **first-line** marker
`// sqlj:experimental-java-view` and correct source roots/dependencies. It is
included in the wizard's repository. Without it, basic SQLJ support remains, but
the enhanced Java view and its checks are not equivalent.

| Feature | Implemented behavior and boundary |
| --- | --- |
| Java inside SQLJ | Java declaration/type/member insight through a shared Java view; missing imports/types and invalid calls are highlighted. Translation is not required for this editor model. |
| Java → SQLJ | Navigation to source classes/methods and modeled iterator/context declarations, including nested declarations. Generated Java in the standard Maven output is redirected to SQLJ where matched. |
| Named iterators | Declared column accessors and modeled runtime API available to completion/navigation. |
| Positional iterators | Type-only declarations and modeled positional runtime API; `FETCH` forms are recognized. No promise of full result-schema/type checking. |
| Binds | `IN`, `OUT`, `INOUT`, names and parenthesized Java expressions; Java lexical scope, shadowing and member resolution. Bare binds also act as output targets where SQLJ syntax requires it. |
| Write targets | Output/INOUT target checks include writable variables, fields and array elements; a computed method result is not a writable target. |
| Indicators | Primitive `short` checks; variables, fields and array elements (including Java index expressions), parenthesized forms and input expressions. Output indicators must also be writable. |
| Header expressions | Context qualifiers and iterator assignment targets use Java expression insight. ConnectionContext/ExecutionContext roles are checked when runtime types are resolvable. |
| Declaration options | Modeled iterator/context options and context constructor families, including `dataSource`; runtime/JNDI configuration is still the application's responsibility. |
| SQL validation | SQL injection delegates to available IDE SQL tooling. Complete DB2 grammar, schema matching and translator semantics are not independently implemented. |
| Rename / usages | References and rename handlers exist for selected host variables. Complete cross-language rename/find-usages coverage is not accepted; inspect previews/diffs. |

Checks can be incomplete while typing or when runtime classes cannot be resolved.
A file without red highlighting is not proof that the translator or database will
accept it. The language guide describes editor coverage, not a replacement SQLJ standard.

## Debugging acceptance

| Capability | Evidence / remaining work |
| --- | --- |
| Breakpoints in original SQLJ | Confirmed in the installed plugin with the DB2 wizard project and embedded SMAP. Executable lines only. |
| Maven preparation before launch | Confirmed in the wizard. Reusable task for imported Maven POMs; customer POMs are not modified automatically. |
| Stepping and locals | Implemented and exercised in controlled real-JVM tests; full DB2/IDE acceptance still pending. |
| Watches, Evaluate, conditions and logpoints | Java-context preparation and regression tests present; live installed-IDE acceptance pending. |
| Session diagnostics | Inspection action implemented; installed-IDE validation pending. Reports metadata, not variable values. |
| Java/Spring without Boot and Liberty | Framework-independent design and setup instructions. Dedicated main-class/Liberty deployment/attach acceptance pending. |
| HotSwap, dev-mode rebuilds, arbitrary customer translators | No general guarantee. Rebuild matching metadata and restart the debug session for the baseline. |

See [debugging setup](debugging-project-types.md) and the
[manual checks](debugging-checks.md). Successful JDI or PSI tests alone do not
establish that the complete installed IDE workflow works.
