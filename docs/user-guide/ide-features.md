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
3. Open your SQLJ source. Java integration is automatic for `.sqlj` files,
   including files that begin with copyright/license comments or a package
   declaration. No special comment or per-file activation is required. Existing
   `// sqlj:experimental-java-view` comments are harmless and may be removed.
4. Wait for import/indexing. Confirm imports match the actual package, then test
   Ctrl+B and member completion. Missing runtime dependencies can limit both Java
   API resolution and the plugin's type checks.
5. Keep translation in your build. Generated files must correspond to source;
   do not edit generated Java as the authoritative source.

## Navigation, completion and errors

Java integration supplies type/member completion and error checks in
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
SQL dialect assistance does not configure your translator, binding or deployment pipeline.
Customize SQLJ colors under **Settings → Editor → Color Scheme → SQLJ**.

## Configure your translator and runtime

**Settings → Languages & Frameworks → SQLJ & DB2 Toolchain** exposes project-level
translator, JDBC, generated-root and environment settings. Environment profiles
are defaults, **not container provisioning or server adapters**. The
Testcontainers entry does not create/manage test containers. There is no Test
Connection button or complete automatic build-on-save/IDE compiler pipeline.

Keep translation, binding and packaging in your application's existing Maven or
other build. Configuring this page does not enable build-on-save, translate every
file during an IDE Build, or show translator errors automatically in the editor.
The online-checking fields are translator options, not continuous live schema
validation. For Maven debug integration, follow the
[build tool guide](build-tool-integration.md).

Generated-root registration can add an existing configured directory to the
project model. It does not create generated code or replace Maven root management.
The wizard's Maven profiles operate independently of these settings.

## Credentials

The plugin has no database-password field and does not store passwords in
`.idea/sqlj.xml`. Keep passwords/tokens out of JDBC URLs, additional compiler
options, Maven `-D` properties, POMs and shared run configurations. The Toolchain
page rejects inline credential options and credential-bearing URLs.

For online translator checks, configure **Private SQLJ properties file** under
**Settings → Languages & Frameworks → SQLJ & DB2 Toolchain**. The runner passes
only `-props=<path>` to the translator. Connection settings in that file take
precedence over the page's URL/user/schema fields: when a file is selected, the
runner does not override it with those fields. Enable online checking explicitly.
Offline translation needs no database password.

Create the file outside the project and shared/synced folders, in a directory
readable only by your account. On Windows, create a private directory before
editing the file (PowerShell):

```powershell
$privateDir = Join-Path $env:LOCALAPPDATA "SQLJ\credentials"
New-Item -ItemType Directory -Force $privateDir | Out-Null
$accountSid = [System.Security.Principal.WindowsIdentity]::GetCurrent().User.Value
icacls $privateDir /inheritance:r /grant:r "*${accountSid}:(OI)(CI)F"
if ($LASTEXITCODE -ne 0) { throw "Cannot restrict credential directory access" }
notepad (Join-Path $privateDir "online.properties")
```

Enter your approved translator's properties locally, for example:

```properties
sqlj.url=jdbc:db2://db.example:50000/APP
sqlj.user=app_developer
sqlj.password=REPLACE_LOCALLY
```

The file contains plaintext secrets: the protection is its access permissions,
not encryption by this plugin. On Unix, use a private directory (`0700`) and file
(`0600`); in CI, provision a restricted file from your secret manager for the job
and remove it afterwards. Use the vendor's property syntax/encoding for your
translator version. Do not copy its contents into commands, support reports or
verbose build output. If you use the runner inside a container, provision the
file privately in that container and enter its container path; a host path is
not automatically mounted or copied. Do not bake credentials into images.
Existing enterprise builds can retain their approved credential mechanism.
The generated demo translates offline and uses separate local-only demo database
credentials; replace those before adapting it beyond local evaluation.

## Diagnose editor resolution

Open the affected file, place the caret on the exact reference and run
**Tools → SQLJ: Inspect Navigation at Caret**, or find that action with
Ctrl+Shift+A. Check source-content/root, imports, the file provider, resolved
reference and indexed SQLJ classes. With Java integration enabled, the provider
is `SqljJavaFileViewProvider`. A failed reference with a wrong import needs an import
fix, not a translator run. Use **SQLJ: Inspect Debug Session** for runtime mapping
issues; it requires a paused Java session.

## Build and debug

New wizard projects include **SQLJ Demo (Maven + SMAP)**. For an existing imported
Maven project, add **SQLJ: Prepare Debug Build** last in Before launch and configure
the real POM/goals. Its goals must translate, compile and embed matching debug
metadata. The task does not install those build steps in your POM or deploy servers.
Follow [debugging setup](debugging-project-types.md) for details and current limits.
