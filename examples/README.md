<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Manual integration scenarios

These files are editor fixtures, not a standalone Maven project. Open them in a
sandbox project with a JDK and the real SQLJ runtime dependencies. They intentionally
live outside the plugin's source roots. For executable database checks, generate
the wizard project and transfer a scenario into that project's SQLJ source root.
No production database is required for editor-only checks.

## First checks

1. Open `editor/ReceiverCompletion.sqlj`. Remove a method name after a dot and
   invoke Basic Completion. Verify both expected AND forbidden proposals listed
   in the source. Restore the call after each check.
2. Open `editor/HostScopes.sqlj`. Navigate from :employeeId to the parameter and
   from :name to the local declaration. Check hidden/declaredLater are absent.
   This is a target acceptance scenario; scope behavior is not fixed in step 1.
3. In a copy, leave the SQL unfinished and check completion/navigation before and
   after it. Recovery is a target, not a currently verified capability.

Record plugin commit, exact IDE build, JDK, database plugin availability, scenario,
action, expected result, actual result and any IDE exception. A source file opening
successfully is not evidence of semantic completion or successful translation.

## Coverage backlog

| Scenario | Required checks | Current evidence |
| --- | --- | --- |
| Receiver types | Separate iterator APIs and ordinary Java members | Fixture + automated regression tests added; not run here |
| Host scopes | Shadowing, nested blocks, declaration order, fields | Fixture added; implementation still incomplete |
| Java APIs | Imports, overloads, generics, inherited/chained members | Architecture spike pending |
| Named iterators | Accessors, types, SQL result binding, lifecycle | Partial synthetic model |
| Positional iterators | FETCH/INTO, endFetch, output targets | Dedicated executable scenario pending |
| Bind modes | IN, OUT, INOUT; procedure arguments; writable outputs | Dedicated executable scenario pending |
| Indicators / index forms | Null indicators, arrays/index expressions, customer-specific syntax | Confirm exact syntax and translator; no completeness claim |
| Contexts | Default/named connection and execution contexts | Dedicated scenario pending |
| DB2 SQL | Joins, aliases, CTEs, functions, schema and platform variants | Data-source/dialect verification pending |
| Java interoperability | Handwritten Java calling SQLJ classes before/after build | Packaged IDE scenario pending |
| Debugging | Java/query breakpoints, stepping, locals, watches, remote attach | Live JDI scenario pending |
| Artifact variants | Container translator and externally built customer artifacts | Capture source/class/debug-metadata pairs |

Do not invent customer-specific syntax to fill this matrix. Add small independent
fixtures as each construct is checked against its actual toolchain. Keep the
wizard example as the runnable smoke test, rather than growing it into one large
example that obscures individual failures.

## Architecture prototype

See [the opt-in Java view probe](java-view/README.md) for native Java completion
and shared-document checks. The existing fixtures keep the legacy view.
