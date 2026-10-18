<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Error highlighting acceptance scenarios

These intentionally invalid editor fixtures must not be included in a build.
Copy HighlightHelper.java and JavaErrorProbe.java into
`src/main/java/examples/highlighting`, and SqljErrorProbe.sqlj into
`src/main/sqlj/examples/highlighting`, in a test module with a configured JDK.
Use separate Java and SQLJ source roots: ordinary Java semantic highlighting
under src/main/sqlj is not an acceptance requirement. The installed IDE showed
no Java errors there even with this plugin disabled, whereas src/main/java
reported them immediately. Keep the declared package `examples.highlighting` and retain the marker in
SqljErrorProbe.sqlj. If changing packages, update BOTH helper references and the
intentionally wrong import; moving the file alone does not change its package.

Ensure IntelliJ is not indexing and that the editor's highlighting level is
“All Problems”. Record IDE build, plugin build and enabled SQL/database plugin.
Use the configured customer IDE as well as the sandbox SDK; green tests on the
sandbox SDK alone do not establish compatibility with the installed IDE.

| Scenario | JavaErrorProbe.java | SqljErrorProbe.sqlj | After correction |
| --- | --- | --- | --- |
| Bad import | Red mark on unresolved import | Same | Fix to examples.highlighting.HighlightHelper; error disappears |
| Unknown type | Red mark on MissingHighlightType | Same | Replace with String; error disappears |
| Unknown method | Red mark on doesNotExist | Same, after SQL block | Replace with count; error disappears |
| Wrong parameters | Red mark on count(123) | Same | Remove 123; error disappears |
| Valid surrounding Java | count() before and after invalid lines resolves | count() before and after #sql resolves | No remaining Java errors |

First observe each error, then fix it in the same editor WITHOUT recreating the
file. Repeat once with unsaved edits, once after save, then after closing/reopening
both probes. Add a new misspelled method after the SQL block to confirm analysis
continues updating. No SQLJ translation or database connection is required.
Warnings such as unused fields/imports are not the ERROR marks tested here.

## What is implemented

Native Java analysis remains responsible for ordinary .java files. The shared
SQLJ Java view has a dedicated highlight visitor which reuses IntelliJ's Java
semantic checks on that root. It ignores SQLJ/SQL PSI and uses original document
ranges, rather than inventing its own type/method validator. The bridge is not
active for legacy .sqlj files without the marker, and it does not override the
user's highlighting-level settings.

Automated tests exercise daemon highlights, severity and token ranges, plus error
removal after document edits and a valid Java method around a SQLJ statement.
They configure the JDK running the tests, so missing SDK types are not confused
with SQLJ integration failures. Runtime SDK version and plugin baseline remain
unchanged.

## Explicit limits

This is Java error analysis, not complete SQLJ-aware data-flow or DB2 schema
validation. Java still does not see reads/writes inside hidden SQLJ fragments.
SELECT ... INTO assignment, bind-type/direction errors, host-expression analysis,
SQL-thrown exceptions and SQLJ-aware refactoring need their own semantic model.
Do not interpret this change as proof that those cases are free of false positives.
