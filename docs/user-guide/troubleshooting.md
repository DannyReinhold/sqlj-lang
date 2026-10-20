<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Troubleshooting and support

Check the [capability matrix](compatibility.md) for server/toolchain requirements
and supported integrations. You do not need private source
access to investigate or report an installed-plugin problem.

## License and distribution

Use **Tools → SQLJ: License Information**. Marketplace packages require a valid
product license or trial; an IDE subscription alone does not license this plugin.
If status is INITIALIZING, allow IDE licensing to finish and refresh the dialog.
UNLICENSED means no valid product confirmation is available. Check the account,
license/trial entitlement and approved IDE licensing connection. A covered fallback
release may differ from the newest installed release. CONFIGURATION_ERROR calls
for reinstalling the approved package and reporting its version/hash.

A customer distribution shows its customer reference and uses the written grant,
not Marketplace checks. Do not share license stamps/activation keys in support
reports. Opening files and existing run configurations must not delete or change
source because a license is unavailable; activate and reopen the project if its
license status has changed. The Java file structure itself no longer depends on
activation timing.

## Editor and project import

| Symptom | Check |
| --- | --- |
| `.sqlj` opens as plain text | Plugin enabled, correct file-type association, restart after installation. |
| Java errors/completion missing inside SQLJ | Complete [Java integration setup](ide-features.md#prepare-an-existing-project): source roots, module SDK, imports and SQLJ runtime dependencies. Java integration is automatic; no source comment is required. |
| Java consumer cannot resolve SQLJ class | Match package/import/class names, source content, scope and duplicate names; run the navigation diagnostic on the exact reference. Translation is not required for the enhanced editor model. |
| Plain Java has no errors in `src/main/sqlj` | Keep ordinary `.java` files in the build's `src/main/java` root and reload the build model. Use `src/main/sqlj` for `.sqlj` sources. |
| SQL tables/columns do not complete | Optional Database Tools/SQL support, entitlement, DB2 dialect availability, development data source and SQL resolution scope. The plugin does not auto-connect. |
| Maven tool window is empty | Enable Maven, Add as Maven Project on `pom.xml`, reload and wait for import. |

Open the affected reference and run **Tools → SQLJ: Inspect Navigation at Caret**
(or Ctrl+Shift+A). Confirm source/root, provider, resolution and indexed classes.
Check package/import names against the class you expect to resolve. Correct the
project configuration, reload the build and retry. If the problem remains, include
a redacted diagnostic report with your support request.

## Demo and translator build

| Symptom | Check |
| --- | --- |
| DB2 initialization still running | Follow DB2 logs from the generated project directory. Downloads and SAMPLE creation depend on local resources. |
| Readiness timeout | `docker compose ps`, `docker compose logs --tail 100 db2`; inspect health output, engine resources, port/name conflicts and actual database connection/query success. Do not delete volumes to retry. |
| Container is running, app still waiting | Running/open port is not equivalent to a ready database. The healthcheck connects and executes `VALUES 1`. |
| Translator not found or unexpected selection | Read the `[SQLJ]` build message. New starters select an explicit executable/JAR, then PATH, the conventional local Maven JAR, or container extraction. Fix an invalid explicit path; it does not silently fall back. |
| Added SQLJ file is not translated | New starters translate every `.sqlj` below `src/main/sqlj`. Keep ordinary Java in `src/main/java`. Earlier generated projects retain their original POM; update their build explicitly or generate a fresh demo. |
| Build fails during SMAP installation | Preserve the first helper error, translator options and generated map. Rebuild cleanly; do not disable validation or copy an old map onto different classes. |

Run these commands from the generated application directory, not the plugin ZIP:

```powershell
docker compose ps
docker compose logs --tail 100 db2
mvn help:active-profiles
```

Use development credentials. Translator/database messages come from the actual
build/runtime; not every translator diagnostic is an IDE highlighting feature.

## Debugger

Launch with **Debug**, not Run. Open **View → Tool Windows → Debug** (usually
Alt+5). Confirm **Mute Breakpoints** is off; Find Action can locate that switch.
Set a normal Java breakpoint in the same process as a control, then an executable
SQLJ statement. A declaration or SQL continuation line may legitimately have no
executable location. A changed breakpoint icon alone does not identify the cause.

Use **SQLJ Demo (Maven + SMAP)** for the current starter. In other Maven projects,
ensure translation, Java debug tables and SMAP installation all run before launch.
`mvn compile` stops before the starter's `process-classes` installation. A later
IDE compiler build can overwrite maps. For a server, inspect the actually deployed
WAR/JAR and restart/reattach to the correct JVM after redeployment.

When paused, open the SQLJ file at the suspect line and run **Tools → SQLJ: Inspect
Debug Session**. `MATCH` verifies the embedded source hash against editor text;
`MISMATCH` calls for matching rebuild/deploy; `ABSENT` means freshness is unverified.
The action can inspect the editor's loaded class even when paused in Java. See
[full debugging instructions](debugging-project-types.md) for metadata limits and
[debugger walkthrough](debugging-checks.md) for stepping/evaluation expectations.

## Send a useful issue report

Contact [support@sqlj-lang.org](mailto:support@sqlj-lang.org). GitHub is a generated
public documentation repository; it cannot accept plugin source contributions.
If you use a public issue, share only a sanitized minimal reproduction.

Include:

- Plugin version, exact IDE build/edition and OS.
- Module SDK, Maven Runner/terminal JDK, and application/server JVM as relevant.
- Whether the problem affects the fresh wizard demo or an existing application;
  DB2 family/version, translator version/options and actual build goals/profile.
- Expected result, observed result and a short sequence to reproduce it.
- Relevant redacted navigation/debug diagnostic, first build exception or IDE log
  excerpt (**Help → Show Log in Explorer/Finder**). State whether normal Java works.
- For debugging: launch/attach type, breakpoint line, metadata/fingerprint result,
  actual deployed artifact and whether recompilation/redeployment occurred.

Do not send customer sources, full logs, passwords, tokens, JDBC secrets or
production values. Use a small invented example and review paths, source snippets,
class names and vendor data before sharing reports. Include a reproducible sequence and the relevant application/build configuration.
