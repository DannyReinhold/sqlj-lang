---
title: DSL debugger and verification lessons
---

<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Debug the original source without inventing execution

[Lessons learned](index.md) · [Documentation map](../documentation-map.md)

## Decide which source coordinates you can prove

Generation introduces a second source file and a third runtime representation.
Line-number similarity is not enough to connect them. Vernac keeps source mappings
for reviewed user-authored bodies and validation/conversion entrypoints, with
source/generated hashes and runtime companion fingerprints. Changed or ambiguous
mappings fall back to Java rather than selecting a plausible but wrong location.

Keep paths relocatable and distinguish source provenance from runtime identity.
The debugger must be able to identify the loaded class and class loader, not just a
local `.java` file with the right name. Unsaved edits after compilation do not mean
the running JVM now executes the new text.

Copied Java bodies, declarative mappings and generated wrappers need different
mapping rules. A copied statement has a user-source range. An implicit default or
infrastructure wrapper does not acquire a fictitious DSL statement just to make
stepping look smooth. Constant-folded expressions can have no executable bytecode
location; breakpoint placement and binding must reflect that.

Vernac's bridge uses Java debugger integration, including a position manager and a
Java-backed line breakpoint type. This reuses JVM execution rather than building a
second evaluator. The concrete mapping format and supported contexts belong to the Vernac
implementation; they are not an implemented SQLJ debugger contract. See the
[SQLJ integration roadmap](integration-roadmap.md) for this plugin’s scope.

## Navigation and evaluation need different contexts

A debugger can stop on the correct DSL line, show locals and evaluate expressions,
yet display those same locals in red in the expression editor. Runtime evaluation
and editor resolution are different paths.

The visible source position belongs in the DSL. Java evaluation and its fragment
editor need the mapped Java method's PSI scope, including parameters, locals and
the actual receiver type. Vernac supplies an evaluation position and code-fragment
factories for both explicit Java and implicit host-file requests. It preserves an
already valid Java context rather than replacing it indiscriminately.

Context must come from the requested source location, not whichever stack frame
happens to be selected globally. Separate methods can contain identical statements;
body-text matching alone cannot identify a scope. Host PSI granularity matters too:
a single undifferentiated leaf can collapse position lookup to the start of a file.

Test Evaluate Expression, Watches, completion, conditional breakpoints and logged
expressions individually. A successful evaluation does not establish editor
highlighting or Ctrl+Space behavior. Logpoints and breakpoint conditions are
standard IntelliJ features; the plugin's job is to give their Java expressions the
correct source/runtime context.

## Breakpoint callbacks can recurse

One Vernac regression caused a `StackOverflowError` when setting a breakpoint.
The breakpoint type's source-position callback called the breakpoint's own
source-position accessor, which called the callback again.

The remedy was to preserve the normal DSL source position without requesting it
recursively in that callback. Evaluation obtains Java context through its separate
mapping path. A regression test uses an accessor that fails immediately if called.

General lesson: when implementing an extension callback, establish whether the
accessor you are about to call is implemented *through that same extension point*.
This applies to references, scopes and renderers as well as breakpoints.

## Make generated code less distracting without hiding behavior

Read/Write/Access views are real access boundaries. A debugger may label them more
clearly and expose useful state, but must not pretend the facade is the domain
object or give a read-only projection a mutation operation.

Vernac's view inspection accepts a conservative subset of verified getter bytecode
and reads backing fields through JDI. It does not invoke arbitrary getter methods
while expanding a tree. Computed getters and unsupported shapes fall back to
normal presentation. Optional storage projections are labelled as such. This avoids
executing business behavior merely because a user opens `self`.

Similarly, filter only verified trivial facade accessors when stepping. A blanket
filter for the generated package can skip validation, event callbacks or other
business-relevant execution. Retain explicit/exception breakpoints and actual
thread/stack structure. Deferred delivery really executes later, on a worker: do
not manufacture a continuous emitter stack.

A contributor mode that shows generated Java remains useful. It should disable
presentation conveniences without changing application behavior. Test both modes.
Missing Set Value on a final VO field is not automatically a plugin defect; preserve
actual JVM/type constraints. Editing ordinary mutable locals is a separate case.

## Use a verification ladder

| Evidence | Establishes | Does not establish |
| --- | --- | --- |
| Compiler/model tests | Signatures, source ranges, visibility, recovery | Native editor behavior |
| Generated-Java compilation | The projection/API is legal Java | IDE indexing or lookup insertion |
| Exact target-SDK compilation | APIs and overrides are available | Correct extension registration at runtime |
| Plugin component tests | Registration, predicates and isolated policies | A running IDE's completion/evaluator |
| Real-JVM/JDI tests | Executable locations, loaded fingerprints, locals, field reads | IntelliJ gutter, popup or step UI |
| Packaged installed-IDE test | The exercised user gesture and distribution work together | Every IDE version or unrelated context |

Use each layer for the failure it can detect. Do not call a skipped PostgreSQL test
a pass, a JDI probe a Spring transaction test, or SDK compilation an end-to-end IDE
test. Preserve a small runnable example with observable behavior for manual
acceptance. The [debugging tutorial](../user-guide/ide-features.md) is both a learning
path and such an acceptance guide.

For a release, test the packaged plugin after restart, not only classes loaded in
a test harness. Check compiler/runtime dependencies in `plugin/lib`, keep the LSP
server distribution/classloader separate, and avoid bundling platform libraries.
Recheck plugin unload/disposal and target-IDE compatibility. Deprecated API warnings
are upgrade signals, not something a passing compiler build disproves.

## A compact acceptance matrix

For each supported body kind, exercise the following across LF/CRLF and an unsaved
edit followed by rebuild:

1. Create, remove, disable and re-enable a breakpoint; distinguish unbound locations.
2. Stop, step into/out/over and navigate back to source.
3. Inspect locals and `self`; expand views without executing hidden behavior.
4. Evaluate an expression, complete a local name, add a Watch and set a condition.
5. Use a non-suspending logpoint, then resume through normal failure handling.
6. Modify source after compilation and check conservative stale-map behavior.
7. Enter handwritten Java and return; compare contributor mode.

Test concurrent/deferred contexts with their genuine thread and transaction
boundaries. Remote attach, module source attachment and Hot Swap need additional
acceptance scenarios; source mapping alone is not evidence for them.

## Wizard DB2 cold-start acceptance

> [!WARNING]
> **FIRST DB2 START: ALLOW MANY MINUTES — PLEASE DO NOT ABORT EARLY.**
> Downloading the image and creating SAMPLE can take a very long time. The starter
> allows **15 minutes for readiness after container startup**, plus image download
> and build time. **The complete first launch can exceed 15 minutes.** A running
> container is not yet a ready database. Keep the build running and use
> `docker compose logs --follow db2` in a second terminal to see initialization.
> Later starts normally reuse the database and become ready much sooner.

Generate a **new** starter after installing the updated plugin; existing starter projects
are not rewritten. Test the first initialization in an isolated Docker environment without
removing an existing database. Run `mvn spring-boot:run` and keep the complete output.

- Before container startup, Maven explains that initialization may take minutes and prints
  the readiness budget and a command to follow DB2 logs in another terminal.
- Maven uses `docker compose up --wait --wait-timeout 900 db2`. Extraction and SQLJ translation
  must not run after a failed health check. Health requires SAMPLE connectivity and `VALUES 1`.
- After Java starts, an immediate notice and a 15-second elapsed-time heartbeat remain visible
  until the application reports ready or failed. The heartbeat covers application initialization,
  including Flyway; it does not claim that DB2 has reached a particular initialization stage.
- Both Maven and Spring allow 15 minutes for readiness. Image download/build time is additional.
  Health-check failures have a 10-minute initialization grace period; successful checks can
  make a warm database ready immediately.
- On success, check the page at `http://localhost:8080` and verify seeded products. Restart without
  recreating the container and confirm readiness is detected promptly, without a fixed sleep.
- With Docker stopped, confirm Maven fails visibly and does not extract/translate. For timeout
  handling use a short readiness budget in an isolated test environment; preserve the logs and
  inspect `docker compose ps` and `docker compose logs --tail 100 db2`.
- Confirm heartbeat output stops after success and failure. With
  `spring.docker.compose.enabled=false`, the Java reporter stays silent; Maven's separate
  container translator fallback is still active unless a local translator is available.

Configuration: `-Ddb2.startup.timeout.seconds=1200` changes Maven's Compose wait budget;
`spring.docker.compose.readiness.timeout=20m` changes Spring's independent readiness budget.
Neither setting changes the image health-check grace period/retry policy automatically.

Reference: [Compose up wait options](https://docs.docker.com/reference/cli/docker/compose/up/)
and [Spring Boot Docker Compose properties](https://docs.spring.io/spring-boot/3.4/appendix/application-properties/index.html#appendix.application-properties.docker-compose).

## First SQLJ runtime mapping patch

The position manager now asks JDI for explicit source strata and only displays SQLJ
coordinates that the loaded class advertises. A generated `Dao.java:103` is never
assumed to mean `Dao.sqlj:103`. Package/name ambiguity, conflicting strata and source
lines outside the current document defer rather than select an arbitrary SQLJ file.
The new wizard pipeline checks embedded source fingerprints as described below,
including edits that preserve line counts. Arbitrary bytecode changes and different
class loaders with the same name are not authenticated by a source fingerprint.

The `SqljDebugLocationsTest` suite checks missing/invalid source information, explicit
strata, native SQLJ SourceFile metadata, exact breakpoint lines and ambiguity.
`SqljDebugJdiTest` compiles a small fixture with debug info, attaches a test-only SMAP,
launches a local JVM, sets an actual SQLJ breakpoint and checks the stopped local value.
It uses a JDK and loopback JDI connection, with a bounded event wait and process cleanup.
It is independent of DB2 and does not prove an installed IntelliJ breakpoint session.

For the generated wizard build, both translator profiles request `-smap=yes`.
After installing this plugin and generating a new starter, run `mvn clean package`
and preserve the generated `.java.smap` files, generated Java and compiled classes.
Do not edit generated lines or replace Java line numbers manually. A separate SMAP
file alone does not enable JVM strata: the generated build helper below performs
post-javac embedding. Until the actual container/installed-plugin acceptance passes,
ordinary generated-Java debugging remains the fallback.
Existing generated applications are not rewritten by installing a plugin update.

Reference: [IBM SQLJ translator options](https://www.ibm.com/docs/en/db2/11.5.x?topic=preparation-sqlj-sqlj-translator),
[JDI source strata](https://docs.oracle.com/en/java/javase/21/docs/api/jdk.jdi/com/sun/jdi/Location.html)
and [JDI source-line lookup](https://docs.oracle.com/en/java/javase/21/docs/api/jdk.jdi/com/sun/jdi/ReferenceType.html).
The JDI documentation is used for the JVM API, not as an Oracle SQLJ integration.

## Automatic wizard SMAP installation and source freshness

The starter now writes `tools/SqljDebugInfo.java` under MIT. It runs as a standard
JDK source-file program with no dependency on the installed IntelliJ plugin.
`initialize` snapshots canonical UTF-8 SQLJ source hashes and removes old sidecar
maps. Both translation profiles request `-smap=yes -linemap=no`; Maven then compiles
with source, line and local-variable debug information. At `process-classes`, the
helper validates the fresh sidecars and installs SourceDebugExtension in matching
package/SourceFile classes, including nested classes. Ordinary Java SourceFile,
line tables, locals and bytecode are preserved; Java remains the default source stratum. Running install twice is idempotent.

The current validated input shape is one stratum, one SQLJ source and standard
JSR-45 file/line tables per map. Repeated/incremented line entries are supported.
Unknown/missing maps, ambiguous source names, source edits during the build,
out-of-range maps, missing compiled classes and classes older than generated Java
fail visibly before modifying class files. This is a trusted-build pipeline, not
cryptographic authentication of arbitrary class files or proof of a translator's
semantic correctness. Actual DB2 map-format compatibility still needs the container run.

A vendor section in the loaded class's SMAP stores the source path and SHA-256.
The plugin compares that hash with current editor text, normalizing BOM/newline
style. A mismatch rejects SQLJ breakpoint/source mapping, including same-line-count
edits; rebuild with Maven. Older native SQLJ metadata without this vendor section
remains usable but has no source-freshness guarantee. Generated files alone are
never used to assert which bytecode is currently loaded.

Use `mvn clean package`, `mvn process-classes`, or `mvn spring-boot:run` in a **new**
wizard project. The Boot run goal forks through test compilation, which includes
process-classes. `mvn compile` and direct compiler goals stop too early. For an
IntelliJ Debug launch, execute Maven through process-classes first and avoid a
subsequent Java-only rebuild. Automatic IDEA before-launch wiring is still pending;
this patch is not the final out-of-the-box debugger acceptance gate.

The JVM smoke test now invokes the external javac from Gradle's configured JDK
compiler toolchain. It no longer depends on `ToolProvider.getSystemJavaCompiler()`
inside IntelliJ's test classloader. Its actual generated build helper installs the
fixture map; a child JVM then hits SQLJ line 12 / Java line 5 and exposes local value
21. Tests cover nested classes, untouched Java classes, idempotence, invalid/missing
maps, changed input, old classes, Windows-style newlines and paths containing spaces.
This remains a controlled fixture; the DB2 container/installed-plugin run is separate.

Reference: [SourceDebugExtension class-file attribute](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.7.11).

## Wizard regression: healthy DB2 rejected by cleanup

The installed-container evidence showed successful `CONNECT TO SAMPLE` and
`VALUES 1`, followed by SQL1024N/exit 4 from the separate `connect reset` invocation.
Compose consequently never became healthy even though the query worked. The new
healthcheck sends CONNECT, VALUES, CONNECT RESET and TERMINATE to **one** CLP
invocation (`db2 -st`). Semicolon termination and stop-on-error keep real connection
and SQL failures visible without relying on session state across frontend invocations.
The shell contract test reproduces the former cleanup failure and checks success,
connection failure and query failure for the new batch. It is a controlled CLI
substitute, not a DB2-container acceptance run.

Existing generated projects can repair the exact old healthcheck from the plugin
checkout with `py -3 scripts/repair_starter_healthcheck.py --project PATH_TO_STARTER`.
The script keeps a byte-for-byte backup and preserves other settings/line endings;
unknown/custom checks are refused. Reapply compose.yaml with `docker compose up`
from that starter project; changing the plugin alone does not change an existing
container's health configuration. No volume-deletion command is needed.

New wizard projects inherit the project SDK, register Java/SQLJ/resource roots,
enable the shared Java view in ProductRepository.sqlj and retain the SQLJ source
root in Maven's build-helper configuration. After project opening and the root-model
write, the optional Maven service registers pom.xml via the bundled SDK API. Check
that the Maven tool window contains the generated pom and dependencies are imported.
A disabled Maven plugin produces an explicit instruction to enable it and attach
pom.xml manually. Core SQLJ support continues to load without Maven.

Navigation acceptance: Ctrl+B on ProductRepository in ProductController must open
the SQLJ class name before translation and after generated Java appears under
`target/generated-sources/sqlj`. Generated-class redirection matches qualified
names inside the same content root, never translated text offsets. Duplicate
SQLJ declarations are refused. This slice handles generated **class** targets;
member redirection from generated Java and arbitrary output directories remain
separate work. Ordinary Java navigation is unchanged.

Reference: [IBM CLP options](https://www.ibm.com/docs/en/db2/11.5.x?topic=clp-options).

## DB2 compressed SMAP output ranges

The container translator emitted `21,6:21,66`: six SQLJ input lines starting at
21, each covering 66 output coordinates. The earlier helper rejected the whole
range against physical generated-Java EOF. That was too strict: unused tail
coordinates do not create executable bytecode locations. Installation now preserves
such ranges unchanged, while retaining input-source bounds, positive fields and
signed-int overflow checks for JDI. It does not clamp or guess source positions.

Regression tests include the exact reported rule, rejected overflow/zero increment,
and a real child JVM with a 66-line output block extending beyond an eight-line
Java fixture. The SQLJ breakpoint still hits Java line 5 and local value 21 is
available. Existing per-location source/line/fingerprint checks remain in effect.
To update an existing generated project, copy the corrected template from
`src/main/resources/wizard/SqljDebugInfo.java.template` to that project's
`tools/SqljDebugInfo.java`, then run Maven through process-classes or package.

## Installed IDE: registering SQLJ breakpoint requests

A custom XDebugger breakpoint type does not inherit the Java handler merely by
extending JavaLineBreakpointType. The reference SDK dispatches handlers by exact
breakpoint type. SQLJ now registers a JavaBreakpointHandlerFactory for its exact
type, retaining the Java debugger's native request/enabled/disabled lifecycle.
The SQLJ type takes priority over Java for a shared Java view, uses all-position
line properties and suppresses Java-only lambda/inline-return variants. Mapped
locations are accepted only for the same original file and line; the Java PSI-only
method-position check is not applied to SQLJ base PSI.

Platform regression tests check extension discovery, the exact dispatch key, type
priority, actual XDebugger breakpoint storage, native Java LineBreakpoint creation,
original SQLJ source position and same/different-file/line acceptance. They do not
replace an installed-plugin JVM session. The standalone JVM mapping tests remain
a separate layer and never proved this IDE registration path.

Installed-plugin acceptance:

1. Build/install the plugin, restart the IDE and start a new debug session.
2. Recreate the test breakpoint in SQLJ so it uses the SQLJ type and line properties.
3. Select the wizard's **SQLJ Demo (Maven + SMAP)** configuration. Older starters
   can add it using `scripts/install_starter_debug_configuration.py`. The general
   plugin task **SQLJ: Prepare Debug Build** runs Maven before launch, including
   compilation and SMAP installation. A Java-only Build must not run afterward.
4. Debug SqljDemoApplication using that configuration. Pick an
   executed Java statement in the SQLJ constructor or repository method first;
   trigger the repository query by opening the product page for a query breakpoint.
5. Check that the breakpoint becomes verified and execution stops at the original
   SQLJ line. Test both a breakpoint installed before class loading and one added
   after the class is loaded. Also test disable/enable, conditions, stopping and
   restarting the session; evaluation/stepping acceptance is still pending.

If it does not bind, capture the breakpoint's displayed type/status, the actual
run configuration and the loaded class's SQLJ strata/fingerprint. Local source
maps alone do not establish which class bytes the IDE launched.

### DB2 maps with overlapping expanded blocks

The starter build helper recognizes the DB2 block pattern observed in the container
translator output: for example, `21,6:21,66` followed by `28,13:87,1`.
A JVM expands the first rule through output line 416, hiding the following explicit
rules. The helper embeds `21:21,66` for that block instead, so the following rule
can map Java line 89 to SQLJ line 30. The original `.java.smap` sidecar is retained.
Only this recognizable trailing overlap is normalized; other overlapping patterns
fail the build before class files are modified. Non-overlapping JSR-45 rules remain
unchanged. A real-JVM regression exercises the complete reported DB2 map.

Expanded SQL blocks map to their first source line. SQL continuation lines and
iterator declaration lines do not thereby acquire separate executable locations.
Set a breakpoint on the opening `#sql` line or on an executable Java statement.
Existing starter projects need the updated `tools/SqljDebugInfo.java` and a clean
Maven package build before starting a new debug session.

### Reusable before-launch build preparation

`SqljDebugBuildTaskProvider` registers **SQLJ: Prepare Debug Build** for existing
run configurations. Per-configuration state stores a POM path and Maven goals;
the optional Maven bridge delegates to the bundled Maven before-run provider.
That provider saves documents, uses Maven runner settings, waits for completion,
and returns failure to IntelliJ instead of launching an application after failure.
No Maven implementation classes are referenced by the core task/provider.
Other toolchains need future adapters; a project still supplies its own translation
and metadata-installation build steps. This is general launch orchestration, not
a universal translator or automatic retrofit of customer build files.

The wizard writes `.run/SQLJ-Demo.run.xml`, using an Application configuration so
both Community and Ultimate can launch the main class. It sets project working
directory, module and main class, enables SQLJ preparation with `clean process-classes`,
and explicitly disables ordinary Make. The installation script reuses the same
MIT template for older starters. It never changes other existing configurations.

Manual acceptance: launch without a separate terminal build; stop at executable
Java and opening SQLJ lines; stop, edit/save source, relaunch and verify the new
code is executed with fresh mappings. Introduce a compilation error and verify
that no new application is launched. Also check missing Maven import, renamed
modules and a wrongly ordered Build task. SDK tests cover provider registration,
state roundtrip, ordering rules and wizard output. Python tests cover installation,
XML escaping, idempotence and preservation of existing/custom configurations.


### Mapped stepping and suspended-frame locals

The former extra stepping filter always applied and returned STEP_INTO, while its
runtime-location predicate was never consulted. The implementation now inspects
the suspended frame and returns STEP_OUT only for exact SQLJ/DB2 runtime package
prefixes. Null/disconnected/unavailable frames do not override stepping. User,
Spring and Liberty frames are left to the native debugger and user settings;
synthetic user methods are no longer indiscriminately filtered by this plugin.

`SqljSteppingPolicy` has no IntelliJ dependencies. The IDE filter and a new real-JVM
regression share it. `SqljDebugSteppingJdiTest` installs SMAP using the real starter
helper, starts suspended, consumes VMStartEvent once and uses one-shot JDI step
requests. It checks Step Over across multiple generated Java lines mapped to one
SQLJ statement, Step Into/Out between SQLJ methods, stepping out of a controlled
runtime fixture, parameter/local values and the caller's source position/locals.
The JVM retains Java as its default stratum, and SQLJ coordinates are queried
explicitly. New steps reacquire frames after suspension; stale frames are not reused.

This is a JDI regression, not an IntelliJ stepping-command test or IBM-runtime test.
Use [the installed-plugin procedure](../../examples/debugging/README.md) to record
F8/F7/Shift+F8 results, selected-frame variable display and unexpected generated
stops. Watches/evaluation still require the next dedicated implementation and
acceptance slice; reading JDI locals is not equivalent to expression evaluation.


### Java expression preparation at the suspended SQLJ position

The old code-fragment factory only overrode the deprecated `createCodeFragment`
entry point and selected a whole class as its Java context. The current SDK uses
`createPsiCodeFragment` and its presentation counterpart through final wrappers;
the implementation now overrides the supported `*Impl` hooks. Deprecated public
bridges delegate to those wrappers, rather than calling the deprecated base method
that can throw AbstractMethodError. The native evaluator and runtime wrappers are
retained.

The effective Java context is the projected leaf at the original SQLJ offset.
For a native SQLJ statement, the shared scope resolver uses its retained semicolon
as the Java lexical anchor. Native Java sibling leaves are accepted and preserved;
ordinary Java and null contexts remain untouched. No generated-source lookup or
whole-class fallback is used. Legacy projected views retain their method context.

`SqljCodeFragmentScopeTest` covers parameters/locals, field shadowing and explicit
this, expired/future locals, catch variables, selected-method scope, nested-class
this, static context, native/legacy views, source/debugger imports, presentation
and code-block fragments, legacy entry points and unrelated Java. These are SDK
PSI tests. They require the pinned IntelliJ SDK to run and do not prove live
expression execution. Follow [the Watches/Evaluate procedure](../../examples/debugging/README.md)
for installed-plugin acceptance and record parse/resolve/runtime failures separately.


### Conditions/log expressions and explicit Java factory dispatch

IntelliJ's `DebuggerUtilsEx.getCodeFragmentFactory` filters extensions by the
expression's stored file type and then by accepted context. The SQLJ-only
registration cannot be selected for an expression explicitly labelled Java.
`SqljJavaCodeFragmentFactory` therefore exposes the same lexical preparation with
JavaFileType while retaining the strict SQLJ-context predicate. The original
SQLJ registration remains for inferred and existing SQLJ-labelled expressions.
Neither registration accepts ordinary Java context. The Java registration is
ordered first so the source-specific handler is selected before a generic fallback.

No independent condition/log evaluator is added. The native line breakpoint
already selects a code-fragment factory for condition and log expressions and
executes the native Java evaluator. `SqljBreakpointExpressionTest` covers the
actual factory-dispatch wrapper, explicit/inferred/legacy labels, SQLJ-opening-line
scope, Java sibling roots, ordinary Java isolation, boolean and String expression
PSI, local/field shadowing and unknown names. A real SDK SQLJ breakpoint carries
a Java-labelled condition/log expression through the existing Java bridge, uses
SuspendPolicy.NONE and retains its independent log when the condition is cleared.

These are SDK selection/PSI/settings checks, not runtime evaluator tests.
[The manual procedure](../../examples/debugging/README.md) specifies false/true
hits, log-only execution, combined condition/log filtering and visible error
feedback. The existing JDI regressions do not substitute for that acceptance.


### Shared-file test contexts and inferred expression language

A source leaf obtained through findElementAt in a shared SQLJ view can belong to
PsiJavaFile rather than SqljFile. Tests must obtain the SQLJ root through
`leaf.getContainingFile().getViewProvider().getPsi(SqljLanguage.INSTANCE)` before
casting it, and use the desired root's AST when testing language-specific leaves.
The production factory already accepts both roots through the view provider.

When an expression has no stored language, IntelliJ infers its file type from
the actual context's containing PSI file. Thus a Java-root context selects the
SQLJ-aware Java alias, while a SQLJ-root context selects the original factory.
An explicitly stored SQLJ expression remains supported with either root.
Regression assertions now check both exact factory choices and parameter/local
resolution through the real dispatch wrapper, rather than assuming every leaf in
a `.sqlj` document belongs to its SQLJ root. No runtime factory behavior is changed
by this test correction.

## Paused-session metadata diagnostics

`SQLJ: Inspect Debug Session` separates editor PSI collection under a read action
from JDI access in a `SuspendContextCommandImpl` on the debugger manager thread.
A resumed/stopped session cancels the command and asks the user to retry. No JDI
is accessed by the action's presentation update. The action remains discoverable
without an editor and explains missing paused-session prerequisites.

`SqljDebugRuntimeSnapshot` captures immutable metadata without reading variable
values or invoking application code. Pure tests cover report fingerprints,
missing metadata, unrelated/nested classes, bounded content and source/vendor
payload exclusion. Real-VM breakpoint tests also capture the selected frame,
visible variable types, executable SQLJ locations and matching provenance.
SDK fixture tests cover action registration and SQLJ/Java sibling-root context
selection. These require the pinned IntelliJ SDK and are separate from the
standalone JDI checks. Manual DB2 and Liberty acceptance remains necessary.
