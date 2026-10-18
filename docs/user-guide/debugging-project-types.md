<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# SQLJ debugging: Java, Spring and WebSphere Liberty

The debugger integration uses IntelliJ's Java debugger and the loaded classes'
SQLJ source maps. It does not depend on Spring Boot, Spring annotations or an
embedded web server. The Spring Boot wizard is one configured example.

The same requirements apply to a plain Java application, Spring without Boot,
and a Spring application deployed to WebSphere Liberty (WLP): translation,
compilation and debug metadata must correspond to the source open in the IDE.
The server must actually execute those classes.

## Current acceptance status

| Project / launch | Configuration | Status |
| --- | --- | --- |
| DB2 wizard application | Application; SQLJ preparation before launch | Source breakpoints and automatic Maven preparation confirmed in the installed plugin |
| Java or Spring without Boot, started by a main class | Application; same preparation task | Framework-independent implementation; dedicated project acceptance remains to be recorded |
| Spring on WebSphere Liberty / WLP | Server debug session or Remote JVM Debug | Setup documented below; a real Liberty session has not yet been accepted |
| Different customer translator/build, remote/container deployment | Java debugger plus that build's metadata | Additional integration and acceptance required; no blanket compatibility claim |

Stepping, expression evaluation, class-loader/redeployment behavior and each
server/JDK combination require separate verification. A successful Maven build
alone does not verify a deployed application.

## Build requirements for every project type

The required order is:

1. Translate the relevant `.sqlj` inputs and emit source maps.
2. Compile generated Java with line, variable and source debug information.
3. Embed the corresponding SMAP in the compiled classes, including nested classes.
4. For server applications, package and deploy those classes.
5. Launch or attach the Java debugger and execute the relevant application path.

The wizard performs steps 1–3 with its generated Maven executions and
`tools/SqljDebugInfo.java`. **Adding SQLJ: Prepare Debug Build to a customer project
does not add these executions or replace its translator/binding process.**
Configure the customer's debug build to produce equivalent metadata first.
Binding and deployment settings remain part of that project's toolchain.

For the wizard helper, the installation phase is `process-classes`. Therefore
`mvn compile` stops too early. `process-classes` is sufficient for a direct launch
from the compiled output directory; `package` is needed when deploying a WAR/JAR.
Use `verify` or other project-specific goals if packaging or post-processing
requires them. Maven's `deploy` lifecycle phase publishes artifacts to a Maven
repository; it does not by itself deploy an application to Liberty.

Configure the existing Maven compiler plugin to retain debug information, for
example with this configuration fragment:

```xml
<configuration>
    <debug>true</debug>
    <debuglevel>lines,vars,source</debuglevel>
</configuration>
```

Keep the project's existing compiler version, toolchain and Java release target
appropriate to its server. The plugin's IntelliJ SDK version is a separate
compatibility concern and does not prescribe the server JVM version.

Bytecode enhancement, instrumentation, obfuscation and subsequent compilation
must preserve the embedded source maps. If a later build stage replaces class
files, metadata installation must follow that stage and still match its line
tables. Do not merely attach an old SMAP to unrelated bytecode.

The current starter helper accepts one stratum and one SQLJ input per map.
It is not a general replacement for every customer translator's mapping format.
The source resolver currently requires an unambiguous SQLJ file matching the
top-level class name and package; duplicate fully qualified names in the IDE
project are rejected rather than selecting an arbitrary source.

## IntelliJ setup shared by all launch types

- Enable the SQLJ and Java plugins. Enable Maven support for the Maven preparation
  task and import the selected POM as a Maven project.
- Configure the application's module SDK and IntelliJ's **Maven Runner JRE** for
  the project. A Java version mismatch between terminal Maven, IDE Maven and the
  server can fail a build or produce incompatible classes.
- Import/register SQLJ sources as source content, for example `src/main/sqlj`.
  Keep ordinary Java in `src/main/java`; generated Java has its own generated
  source directory. The wizard's build-helper setup registers these paths.
- Match the IDE checkout to the deployed source revision. Rebuild and redeploy
  after changes; the plugin rejects a known source-fingerprint mismatch.
- Place breakpoints on executable Java statements or the opening `#sql` line.
  Declarations and SQL continuation lines may have no executable location.

The shared Java view is currently opt-in with the first-line marker
`// sqlj:experimental-java-view`, as in wizard sources. This controls Java PSI
integration; it does not generate runtime SMAP metadata.

### Configuring SQLJ: Prepare Debug Build

Open **Run > Edit Configurations**, select the relevant configuration, and add
**SQLJ: Prepare Debug Build** under **Before launch**. If the list is hidden,
use **Modify options > Add before launch task**.

Set an absolute Maven POM path and the goals of the actual debug build. The task
uses IntelliJ's Maven runner/settings, saves editor changes, waits for the build
and prevents launch if the build fails. The POM must already be imported.

Keep preparation after any ordinary IDE Build. A later ordinary Build is rejected
because it could overwrite SMAP-equipped classes. You can disable a redundant
ordinary Build entirely; the wizard does so. Other custom tasks are not inspected
for class replacement: their ordering remains part of the project's setup.

For a multi-module project, select the reactor POM and the necessary module/profile
arguments. For example, a project's goals might be
`-Psqlj-debug -pl :customer-web -am clean package`. These names are illustrative,
not profiles supplied by this plugin. Apply translation and metadata installation
in the modules that own SQLJ sources, including libraries packaged in the web app.

## Java and Spring without Spring Boot

Use an ordinary **Application** run configuration with the real main class,
application module, working directory and runtime dependencies. Add the SQLJ
preparation task with the goals that finish metadata installation. No Spring Boot
configuration or Boot Maven plugin is needed.

The preparation task does not hardcode `SqljDemoApplication`, a package name,
Docker or a database server. Those are defaults of the generated example, not
requirements of the debugger. If Spring is hosted in a servlet/application
server rather than a main program, use that server's launch/attach setup instead.

## Spring on WebSphere Liberty / WLP

Liberty runs the application in its own JVM. The Java debugger must attach to
that JVM; starting a separate application main class would debug a different
process. Open Liberty's command reference documents `server debug`; IBM documents
the corresponding Liberty environment variables. Follow the commands and JDK
requirements of the installed Liberty version (links below).

### Controlled build, deployment and attach

For the first acceptance run, use a local development server and a repeatable
build/deploy cycle:

1. Run the application's Maven debug build through packaging. Verify that the
   class files in the resulting WAR/JAR contain `SourceDebugExtension` and SQLJ
   line information. In a WAR, application classes normally live in
   `WEB-INF/classes`; a SQLJ library can instead be a JAR in `WEB-INF/lib`.
2. Deploy that exact artifact using the project's existing Liberty procedure.
   A local `target/classes` check does not establish what the server loaded.
3. Start the development server with Java debugging enabled. For example, from
   its `wlp/bin` directory on Windows:

   ```powershell
   .\server.bat debug sqljDebugServer
   ```

   Replace the illustrative server name with the configured development server.
   `server debug` runs in the foreground and normally waits for the debugger.
   The documented default debug port is **7777**; `WLP_DEBUG_ADDRESS` can change
   it. Use the actual port from the server output/configuration. This is a JDWP
   port, separate from the application's HTTP port.
4. In IntelliJ choose **Run > Edit Configurations > + > Remote JVM Debug**.
   Select **Attach to remote JVM**, a socket connection, the server host (for a
   local server, `localhost`) and its debug port. Select the application module
   containing the matching SQLJ sources in **Use module classpath** / source lookup
   (the label varies by IDE version).
5. Start this configuration with **Debug**, then invoke the application path that
   reaches the SQLJ method. Confirm an ordinary Java breakpoint first, then an
   SQLJ constructor/method statement and a query's opening `#sql` line.

An IntelliJ application-server configuration can also manage server startup and
deployment when the appropriate server integration is installed. Verify that it
deploys the already mapped artifact instead of recompiling it afterward. A
dedicated WebSphere/Liberty IDE integration is not required for the documented
Remote JVM Debug path.

### What belongs in Before launch for a server

For attach to an **already deployed application**, attach directly. Rebuilding
local sources does not replace classes in that JVM. Leave the preparation task
out of an attach-only configuration unless the configured finite build/deploy
pipeline actually updates the target before attach.

For an IDE-managed **build, deploy, start** workflow, metadata installation must
finish before packaging/deployment, and deployment must finish before executing
the new application. The plugin's preparation task can run the finite Maven build;
it does not automatically deploy WARs, start Liberty or select a server instance.

Do not use a long-running command such as `liberty:dev`, `liberty:run` or a
foreground server-debug process as the preparation task's Maven goals: the task
waits for the command to exit before allowing the debugger launch to proceed.
Use the established server integration or a separate terminal for server lifetime.

### Development mode, redeployment and HotSwap

Liberty dev mode can recompile changed Java and hot deploy changes. Its
automatic cycle must include SQLJ translation and SMAP installation after each
compilation. Otherwise a previously working source breakpoint can stop binding
after an edit. The plugin does not yet integrate the complete dev-mode cycle.

For initial acceptance, avoid concurrent IDE compilation/dev-mode rebuilding of
the same output. Build the mapped artifact, redeploy it and start a new debug
session after edits. Do not assume IntelliJ HotSwap reruns the SQLJ translator or
reinstalls metadata. Loose applications and container volume mounts also need to
point to mapped classes, not another output directory. Confirm the actual class
loader/artifact when multiple versions of a library are present.

## Checking stepping and variable display

Use the [manual stepping and frame scenario](../../examples/debugging/README.md)
with a rebuilt wizard project. It covers F8 over SQLJ blocks, F7 into a user helper,
Step Out, parameters and initialized locals, and switching between caller/callee
frames. A breakpoint stops before the statement executes, so a local initialized
on that line may only appear after the next step. Watches and expression evaluation
are a separate acceptance check.

The SQLJ extra stepping filter only steps out of recognized SQLJ/DB2 runtime
packages. It does not globally force Step Into, hide user lambdas/synthetic methods,
or filter Spring/Liberty frames. IntelliJ's own stepping preferences still apply.

## Watches, Evaluate Expression and lexical scope

The SQLJ code-fragment factory prepares **Java** expressions at the suspended
source position, using the same Java projection as host-expression code insight.
It keeps the enclosing method/block rather than using a class-level context.
Current SDK hooks cover both evaluation and presentation fragments; compatibility
bridges also handle the deprecated entry points.

For the [manual helper scenario](../../examples/debugging/README.md), stop after
`offset` is initialized, select the helper frame and evaluate `count`, `offset`
and `count + offset` in Watches and **Evaluate Expression** (`Alt+F8`). Select the
caller frame to inspect its locals and `this.dataSource`. Enter Java expressions,
not `:count` or `#sql` text. A static frame has no `this`; not-yet-initialized or
out-of-scope locals can legitimately be unavailable.

This patch adds lexical-context preparation and SDK regression coverage. Live
Watches/evaluation, conditions and log expressions still need installed-plugin
acceptance. The native Java debugger remains responsible for expression execution,
actual runtime values and method invocation; the plugin does not introduce a
separate SQL evaluator. The same context preparation has no Spring Boot dependency.

## Conditional breakpoints and log expressions

SQLJ line breakpoints retain the native Java debugger's conditions, log actions
and suspend policies. Open a breakpoint's properties by right-clicking its gutter
symbol or through **Run > View Breakpoints** (`Ctrl+Shift+F8`). A **Condition** is a
Java boolean expression in the stopped method/block's scope. **Evaluate and log**
is a Java expression whose value is printed to the debugger Console. Uncheck
**Suspend** for a log-only breakpoint; a condition can also restrict which hits log.

Use [the deterministic condition/log scenario](../../examples/debugging/README.md):
three helper invocations with values 0, 1, 2, a shadowed local/field and an expected
single conditional hit. It also checks non-suspending logs, combined condition/log
behavior and expression errors. Set the breakpoint after the referenced locals'
initializers. Every matching request can execute the expressions again.

Explicitly **Java**-labelled expressions use the SQLJ-aware Java factory whenever
the source context belongs to a SQLJ file. Expressions inferred from SQLJ source
or previously stored with the SQLJ language keep the original factory. Ordinary
Java files are not intercepted. Evaluation remains the native Java debugger's
responsibility; the plugin supplies the lexical context, not a separate language
or server runtime. Live condition/log behavior is a separate acceptance gate.

## Diagnosing missing SQLJ breakpoints

First verify a normal Java breakpoint in the **same running server/application**.
Then inspect the class from the actual deployed artifact, for example after
extracting it to a temporary inspection directory. For a local Maven output,
replace the illustrative class path below with the affected class:

```powershell
javap -v .\target\classes\com\example\repository\ProductRepository.class |
    Select-String -Pattern "^SourceDebugExtension:" -Context 0,150
```

`SourceFile: ProductRepository.java` is expected for generated Java and does not
mean that SQLJ debug information is missing. Inspect the complete SMAP, rather
than matching every occurrence of `sqlj` in the constant pool.

| Observation | Next check |
| --- | --- |
| No Java breakpoint hits | Debug connection, correct server JVM, deployment, executed request |
| Java works; every SQLJ breakpoint is invalid | Loaded artifact's SMAP, source roots, class/package, source fingerprint |
| Some SQLJ lines bind | Whether the selected line has executable code and a mapping |
| Works until recompilation/redeployment | Which later compiler or deployment step replaced metadata or class files |
| Local class has SMAP but server debugging fails | Extracted deployed WAR/JAR, loose deployment path, class loader and source revision |

For an issue report record IDE/plugin/JDK versions, Liberty version, translator
version/options, launch type, Maven goals, source roots and the affected SMAP.
Report stepping/evaluation behavior separately from successful breakpoint hits.

## Inspect a paused debug session

Use **Tools → SQLJ: Inspect Debug Session** (also available through Find Action)
while a Java debug session is paused. Keep the `.sqlj` editor open and place the
caret on the breakpoint line you want to inspect. If SQLJ breakpoints do not bind,
pause at an ordinary Java breakpoint first. Select the relevant stack frame.
The read-only report can be copied with Ctrl+A, Ctrl+C.

The report distinguishes the selected runtime frame from the class expected for
the editor file. It includes the VM, class-loader identity, Java/SQLJ coordinates,
loaded SMAP line rules, locations for the editor line, source fingerprint and
Java expression-factory/context metadata. Matching top-level classes are checked
even when the selected frame belongs to a controller or another Java class.
Multiple class loaders are listed separately (at most five candidates).

| Report | Interpretation |
| --- | --- |
| `MATCH` | The embedded source fingerprint matches the editor text, normalizing line endings. This is not proof that the entire deployed artifact is current. |
| `MISMATCH` | Editor source and embedded fingerprint differ. Rebuild, redeploy if applicable, and start a new debug session. Unsaved changes also count. |
| `ABSENT` | No plugin fingerprint is available. Source freshness is unverified, even if SMAP exists. |
| Zero locations for the caret line | Check whether the line is executable, then the mapping and loaded source revision. |
| No loaded top-level class | The class may not have been used yet; an executing nested class is reported independently through the selected frame. |
| Missing local-variable metadata | Check compiler debug information and the selected frame; no variable values are required for this report. |

The action commits editor PSI without saving the file. It does not evaluate
expressions, invoke application methods, rebuild, deploy or upload the report.
It collects variable names and types, **not values**, and does not include source
text or raw SMAP vendor payloads. Paths, class and variable names can still reveal
project information; review the report before sharing it. Output is limited to
50 visible variables, 60 numeric SMAP rules and 10 locations per class.
For Liberty, verify the deployed WAR/JAR separately: a class-loader identifier
and SMAP digest do not identify an artifact's filesystem path or bytecode hash.

Manual checks: capture a report at a SQLJ breakpoint, then at a Java breakpoint
with the SQLJ editor open. Edit the SQLJ source without saving and verify that
its fingerprint becomes `MISMATCH`; undo the edit and check again. Also run the
action while the VM is running: it should explain that a paused frame is needed.
Successful report capture does not establish stepping, watch or condition support;
continue those acceptance scenarios independently.

## Reference documentation

- [IntelliJ: attach to local and remote Java processes](https://www.jetbrains.com/help/idea/attach-to-process.html)
- [IBM: customizing the Liberty environment](https://www.ibm.com/docs/en/was-liberty/core?topic=liberty-customizing-environment)
- [Open Liberty: server debug command](https://openliberty.io/docs/latest/reference/command/server-debug.html)
- [Liberty Maven plugin: development mode](https://github.com/OpenLiberty/ci.maven/blob/main/docs/dev.md)
- [Maven: build lifecycle phases](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Maven compiler plugin: compile configuration](https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html)
