<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Editor features and existing projects

For features, requirements and limits, see [capabilities](compatibility.md).
The plugin is an editor/debugger integration; it does not replace the application's
SQLJ translator, SQLJ runtime, binding/customization or database deployment.

## Prepare an existing project

1. Import the real Maven POM or existing build model. Configure the project/module
   SDK and the translator-compatible JDBC **and SQLJ runtime** dependencies. A JDBC
   driver dependency alone is not a complete SQLJ toolchain.
2. Register `.sqlj` sources as source content (`src/main/sqlj` is conventional).
   Keep ordinary `.java` in `src/main/java` and generated Java in its own directory.
   Changes made only in the IDE project model may be overwritten by build reimport;
   declare roots in your existing build where appropriate.
3. Enable enhanced Java integration by placing this marker at the very start of
   each SQLJ file, before package declarations or license comments:

   ```java
   // sqlj:experimental-java-view
   ```

   The provider is selected when the IDE creates the file view. After adding or
   removing the marker, close and reopen the project so the view is recreated.
   The marker affects editor semantics, not SQLJ compilation or runtime SMAP.
4. Wait for import/indexing. Confirm imports match the actual package, then test
   Ctrl+B and member completion. Missing runtime dependencies can limit both Java
   API resolution and the plugin's type checks.
5. Keep translation in your build. Generated files must correspond to source;
   do not edit generated Java as the authoritative source.

Without the marker, SQLJ parsing/highlighting and legacy assistance remain, but
the shared Java view's highlighting, expression and scope checks are not equivalent.

## Navigation, completion and errors

The enhanced Java view supports Java type/member completion and error checks in
regular Java sections. Host names and parenthesized expressions use Java lexical
scope: parameters, locals, shadowing, fields and static context. Ctrl+B navigates
host variables to declarations. Named iterator accessors and modeled context and
iterator members are available before translation, including nested declarations.

Java consumers can navigate back to SQLJ classes and methods. Standard Maven
output in `target/generated-sources/sqlj` has source redirection where a match is
found. Duplicate class/package names, unresolved imports or custom generated
layouts can prevent a unique target. Use the navigation diagnostic below.

Errors include unresolved Java names/calls, selected declaration-option checks,
invalid output targets, primitive-short indicator requirements and resolvable
connection/execution-context roles. Incomplete code or unresolved dependencies
can suppress secondary checks. The translator remains the final build authority.

Reference/rename handlers cover selected cases; **cross-language rename
and Find Usages are limited to modeled references**. Use previews and review the resulting diff,
especially for iterator columns, generated APIs and references in SQL fragments.

## Embedded DB2 SQL assistance

The injector prefers a registered `DB2` language, falling back to `SQL` or
`GenericSQL`. If none is available, no SQL language is injected. Rich SQL syntax,
schema inspections and table/column completion depend on the IDE's optional SQL/
Database Tools plugin and its capabilities/entitlement. Configure a data source
and SQL resolution scope using that IDE's database tooling. Merely enabling this
plugin does not connect to a database or discover production schemas.

Java binds and their indicators are replaced by SQL parameter placeholders for
injection. This does not implement every DB2 SQLJ-specific SQL construct, guarantee
SQL formatting behavior or validate query/result types against the catalog.
SQL dialect assistance does not establish z/OS, IBM i or Oracle runtime support.
Customize SQLJ colors under **Settings → Editor → Color Scheme → SQLJ**.

## Toolchain settings: what they currently do

**Settings → Languages & Frameworks → SQLJ & DB2 Toolchain** exposes project-level
translator, JDBC, generated-root and environment settings. Environment profiles
are defaults, **not container provisioning or server adapters**. The
Testcontainers entry does not create/manage test containers. There is no Test
Connection button or complete automatic build-on-save/IDE compiler pipeline.

The implementation contains a translator command runner, but it is not registered
as a general compiler/build action. Do not assume configuring this page causes
translation, binding or inline translator errors on each build. Use your actual
Maven or other build. Online checking fields prepare translator arguments when
the runner is used; they do not enable continuous live catalog validation.

Generated-root registration can add an existing configured directory to the
project model. It does not create generated code or replace Maven root management.
The wizard's Maven profiles operate independently of these settings.

**Credentials:** the current password field is stored in project settings
(`.idea/sqlj.xml`), not IntelliJ Password Safe, and runner arguments can include
it. Leave it empty for sensitive environments and supply credentials through your
approved build/database tooling. Exclude credential-bearing project files from
version control. See [administration](../administration/deployment.md).

## Diagnose editor resolution

Open the affected file, place the caret on the exact reference and run
**Tools → SQLJ: Inspect Navigation at Caret**, or find that action with
Ctrl+Shift+A. Check source-content/root, imports, the file provider, resolved
reference and indexed SQLJ classes. The experimental marker should produce
`SqljJavaFileViewProvider`. A failed reference with a wrong import needs an import
fix, not a translator run. Use **SQLJ: Inspect Debug Session** for runtime mapping
issues; it requires a paused Java session.

## Build and debug

New wizard projects include **SQLJ Demo (Maven + SMAP)**. For an existing imported
Maven project, add **SQLJ: Prepare Debug Build** last in Before launch and configure
the real POM/goals. Its goals must translate, compile and embed matching debug
metadata. The task does not install those build steps in your POM or deploy servers.
Follow [debugging setup](debugging-project-types.md) for details and current limits.
