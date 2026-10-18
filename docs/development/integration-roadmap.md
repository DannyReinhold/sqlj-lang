<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Integration roadmap — 2026-10-18

Baseline: `5f7eff2`. This roadmap supersedes completion claims in the historical
[development plan](development-plan.md). Registered extensions and passing helper
tests do not establish working editor or debugger features.

## Constraints

- Keep the customer's older IntelliJ baseline; no SDK upgrade in this series.
  Actual configuration is currently IC **2024.3.4**, with since-build 261.24374.
  Reconcile the intended 2026.1.x SDK separately before publishing compatibility claims.
- Support named/positional iterators, IN/OUT/INOUT, assignment targets, connection
  and execution contexts, Java expressions and indicator variables. The user's
  additional term “Indexvariablen” remains a separate requirement until concrete
  syntax establishes whether it means indicators, array indexes or another form.
- Target DB2 LUW, z/OS and IBM i explicitly; platform-specific behavior requires
  evidence from that translator/server combination.
- The wizard is a working executable baseline. Preserve the profile packaging fix.
  Desired discovery order: explicit tool, installed tool on PATH, container extraction.
  Current template instead detects a fixed local Maven artifact before container
  extraction. PATH discovery is a future implementation item, not an existing feature.
- First acceptance target: a fresh wizard-generated project using the DB2 container
  toolchain must build, run and debug out of the box. No customer artifacts or
  cross-translator comparison are prerequisites for the first debugger slice.
- Other translators, customer builds, server variants and remote attach are add-ons.
  When introduced, preserve their artifacts rather than silently rebuilding them
  with the reference translator. Oracle remains after the working DB2 baseline.

## Current delivery order and acceptance gates

Canonical source is now the private Gitea repository. GitHub receives only the
allowlisted documentation export; migration and publishing were manually verified.
The review baseline above is historical, not the new Gitea root commit.

| Step | Current state | Remaining acceptance gate |
| --- | --- | --- |
| 1 — examples and architecture | Shared Java view, navigation/completion, error highlighting and scope probes manually exercised | Keep regressions and invalid-code scenarios in every relevant slice; retain the development opt-in until the supported wizard flow is promoted |
| 2 — Java/SQLJ semantics (current) | Host expressions, OUT/INOUT writability, indicators and header expressions implemented; context role/API and constructor-family patches delivered, acceptance pending | Run the latest patches in the reference IDE with the container runtime; complete iterator APIs, context implements/profile APIs, custom attributes and result/bind compatibility; deduplicate generated symbols and validate module visibility |
| 3 — container debugger artifacts | Runtime stratum mapping and controlled JDI test implemented; wizard requests SMAP; post-compile helper implemented; actual container output/IDE acceptance pending | Inspect matching SQLJ/generated Java/class files, translator options and javap metadata from the wizard's DB2 container only; no customer comparison required |
| 4 — out-of-the-box DB2 debugging | Primary acceptance goal after container artifact investigation | Fresh wizard project: build/run, Debug, real breakpoint hit in .sqlj, Step Over/Into, locals and evaluation; automatic source mapping and stale-artifact handling with no manual mapping setup |
| 5 — DB2 SQL integration | SQL injection and indicator-pair projection exist; database-aware integration remains partial | Complete the container baseline's schema/table/column completion, aliases/types and offline behavior first; z/OS/IBM i and customer-specific toolchain/binder forms are later add-ons |
| 6 — cross-language operations | Go to Declaration is implemented; full usages/rename is pending | Find Usages/Rename across Java/SQLJ with visibility/conflict checks and no edits to generated files |
| 7 — toolchain and release hardening | Wizard executable baseline and private-source/public-docs workflow work | Wizard source roots/runtime visibility and installed PATH discovery before container fallback; no hidden marker or hand setup in the accepted default flow; multi-module/remote/customer variants follow as add-ons |
| 8 — Oracle SQLJ (explicitly deferred) | No Oracle-specific integration work in the current series | Start only after DB2/Java integration and artifact-driven debugging are implemented and accepted; then assess Oracle translator/runtime/binding, dialect and debug metadata separately |

Current code is a growing integration prototype, not a declaration that steps 2–7
are complete. Automated helper tests do not replace installed-plugin testing or a
real debugger session. Java files in a mixed src/main/sqlj root remain a low-priority
IntelliJ integration case; ordinary Java fixtures belong under src/main/java.

## Debugger baseline: wizard plus DB2 container first

The first implementation and acceptance run use only the artifacts produced by
the wizard's DB2 container toolchain. Collect the minimal SQLJ source, matching
generated Java/class files, build options, translator/JDK versions and `javap -v -l`
output for this single baseline. Inspect SourceFile, LineNumberTable,
LocalVariableTable and SourceDebugExtension plus JDI strata/source names, then
choose the source mapping from that evidence. Do not wait for customer artifacts
or compare independent translators before implementing the first working debugger.

A fresh wizard project is the primary user journey. Its build/run flow and Debug
action must resolve runtime libraries, source roots, generated artifacts and SQLJ
source positions automatically. The current shared Java view still requires its
experimental marker; removing that manual requirement from the supported wizard
flow is an explicit acceptance task, not an already completed feature. Breakpoints
must actually hit in .sqlj and stepping, locals/evaluation and stale-build handling
must be checked in the installed plugin. Merely registering a PositionManager or
passing offset-helper tests is insufficient.

Do not guess that SQLJ and Java line numbers match. Use the metadata of the known
container translator and report unsupported mappings clearly. Keep the mapping
boundary separable, but implement only the concrete baseline adapter first.
Alternate translators, customer build artifacts, bind variants and remote attach
are additions after this path works. When Oracle is evaluated after DB2 integration
and debugging acceptance, extend that boundary for its real translator/runtime
metadata. Existing customer artifacts must never be silently regenerated using a
different translator. Database binding and JVM debug metadata are distinct.

## Verification

Run `./gradlew test --tests org.sqlj.lang.completion.SqljCompletionTest`, then
`./gradlew test buildPlugin`. Test the packaged plugin on the agreed customer IDE.
Use [manual scenarios](../../examples/README.md); record IDE/plugin/translator
versions with results. Debugger acceptance requires a live debug session, not only
PositionManager unit tests.

In this review environment Gradle bootstrap is blocked by network access and only
JDK 17 is installed. Added regression tests have not been executed here.

## Step 2 prototype

The [experimental Java view](../../examples/java-view/README.md) is implemented
behind a per-file marker. SQLJ stays the base language; a Java template-data root
shares the original document. Completion copies preserve this provider. SQLJ
islands are removed for Java parsing and reinserted as outer-language elements.
This architecture is a candidate, not yet an adopted production default.

New tests cover native imported-type/generic-chain completion, offset/document
identity, Java reference targets and synchronization after edits. Splitter tests
cover quoted/commented directives, terminators and restart behavior. Existing
completion tests must continue passing without the opt-in marker.

Run `./gradlew test --tests org.sqlj.lang.java.SqljJavaViewTest --tests org.sqlj.lang.java.SqljJavaTemplateLexerTest --tests org.sqlj.lang.completion.SqljCompletionTest buildPlugin`.
The prototype does not alter SDK version, wizard tool discovery or debugger mapping.

### Step 2 follow-up: source-only Java interoperability

Synthetic SQLJ classes now participate in Java declaration processing; their methods
and constructors navigate to SQLJ declarations (column accessors to their columns).
The element finder exposes declared packages and parent packages independently of
physical Java package directories. Per-file class models are cached with PSI-change
invalidation. Regular classes in the experimental view are returned as original
Java PSI, retaining generic signatures and source identity.

`SqljJavaInteropTest` exercises imports, package discovery, native member completion
inside SQLJ and in Java consumers, constructor/method navigation, imported column
types and cache invalidation before translation. See the
[interop manual probe](../../examples/java-view/README.md#iterator-context-and-java-consumer-probe).
This does not claim nested declaration ownership or deduplication against generated
Java. Those and full SQLJ runtime signatures must be checked before production use.

### Nested declarations in the shared Java view

A PsiAugmentProvider exposes SQLJ member classes on their native Java owner.
Qualified lookup distinguishes package types, members and members of nested Java
classes; package enumeration only lists top-level classes. Source ownership is
computed from Java syntax ranges without calling augmented getInnerClasses(),
which would recurse while constructing the file's class cache. Synthetic class
identity remains anchored to the SQLJ source declaration.

Declared access/static modifiers are preserved; Java interface member defaults
are modeled. The synthetic type's parent/context follows the Java owner so imports
and accessibility can use the native model. Native AST traversal keeps Java member
classes discoverable without augment-provider recursion. Method-local declarations
are not exported; full local-type support remains a separate design.

`SqljNestedTypesTest` covers ownership, qualified names, package enumeration,
visibility, same-name members, imported iterator/context completion and navigation.
See the [nested manual probe](../../examples/java-view/README.md#nested-declaration-probe).

### Java error-highlighting gate

`SqljJavaErrorHighlightingTest` uses daemon highlights to verify ERROR severity
and source ranges for unresolved imports, unknown types, unknown methods and
invalid call arguments in .java and marked .sqlj files. Each failure is corrected
in the same document and the absence of stale ERRORs is checked. A valid method
around a SQLJ block must stay free of Java errors.

The daemon selects semantic visitors using the base file. The standard Java
visitor accepts Java import holders, so the SQLJ base does not select it. A
SQLJ-only bridge reuses the SDK's Java HighlightVisitorImpl, analyzes the shared
Java root and ignores elements from SQLJ/SQL roots. Its clone retains the bridge
class. Normal .java files continue to use their native visitor without duplication.
This implementation intentionally depends on the Java analysis implementation in
the existing SDK; packaged-plugin checks on the target IDE remain necessary.

Manual acceptance fixtures and limits are in
[the highlighting guide](../../examples/java-view/highlighting/README.md).
The tests configure the JDK running Gradle's test JVM; no IDE SDK version change
or new external database/toolchain dependency is introduced.
Full SQLJ data-flow (including INTO writes), SQL-aware bind checks and native Java
inspections beyond the semantic visitor remain separate work.

### Host binding lexical scopes

Simple host names now use Java's lexical scope traversal with static-context and
member-access checks. The nearest declaration wins; variables in closed blocks,
other methods or later local declarations are excluded. A successful empty Java
scope is authoritative: no whole-file declaration heuristic is applied, and
cancellation/indexing errors are no longer swallowed by that heuristic.

Reference resolution, reference completion variants and the custom SQLJ completion
contributor use the same model. Own-file declarations map to source identifiers;
inherited/imported fields keep their actual Java/library navigation targets.
Bind modes come from lexer tokens, preventing input/index/output/indicator names
from being shortened as if they were IN/OUT/INOUT directives.

`SqljHostScopeTest` covers shared and legacy Java views, shadowing, declaration
order, loops, catch scope, static methods, inherited field visibility/navigation,
completion variants and mode-prefix identifiers. This does not implement arbitrary
Java host expressions, array/index syntax, indicator relationships, bind type/
write validation or all injected-SQL completion entry points.

## Current progress — 2026-10-18

| Area | Evidence / current status | Next gate |
| --- | --- | --- |
| Shared Java view | Build/tests green; basic native completion verified in installed IDE | Extend semantic coverage |
| Java interoperability | Source-only imports, completion and navigation verified; wrong example imports diagnosed | Generated-source coexistence remains pending |
| Nested iterator/context types | Build/tests green; manual nested-type checks confirmed | Extend coverage as needed |
| Java error highlighting | Java and SQLJ error probes confirmed with separate source roots | Extend checks to Java inside SQLJ host expressions |
| Simple host scopes | Build/tests green; ScopeProbe manually confirmed | Extend to complex Java expressions |
| Remaining SQLJ semantics | Expression reference/type prototype prepared; indicator forms, bind types/directions and complete runtime APIs pending | Separate small acceptance scenarios |
| Source debugging | Existing scaffold; no reliable translator-independent mapping established | Inspect real generated class metadata |
| DB2 schema insight | Injection exists; explicit dialect/data-source/schema behavior pending | Database context integration |
| Refactoring/toolchain/release | Partial scaffolding; full acceptance pending | References, generated duplicates, PATH discovery and IDE compatibility |

### Parenthesized Java host expressions — first step

Shared-view files parse the content of `:(...)` as a Java expression fragment,
using the same native Java statement anchor as host-name scope resolution.
References expose individual source ranges; own-file targets navigate to SQLJ
identifiers and external targets retain their Java source. IntelliJ supplies
method overload resolution and expression types rather than a text-based
member lookup. Tests and [manual probes](../../examples/java-view/expressions/README.md)
cover receivers, calls, nested calls, arrays, lexical shadowing and unresolved
names. Member completion, visible error diagnostics, rename and bind-direction
validation are subsequent steps. The prototype is limited to marked shared-view files.

Ordinary Java acceptance files belong under src/main/java; SQLJ files under
src/main/sqlj. Java semantic highlighting under the SQLJ root is outside the
active roadmap, following the confirmed plugin-disabled comparison.

### Host expression completion and Java diagnostics

The host annotator now runs Java's semantic visitor over the contextual
expression fragment and maps ERROR ranges and descriptions back to the SQLJ
host. No fragment quick fixes are installed. Completion adapts native reference
variants from the completion PSI copy and inserts method parentheses into the
real SQLJ editor. SQL keywords are not contributed inside a parenthesized Java
host. Indexing defers these operations.

`SqljHostExpressionInsightTest` adds daemon checks for unknown names/methods,
wrong argument counts/types, disappearance after corrections and valid nested
array expressions. Completion scenarios cover member/local names, chained
receivers, inaccessible fields and insertion with/without an existing call.
Tests are prepared; the author environment still cannot download Gradle, so
these scenarios require execution on the development machine and installed IDE.
The expression navigation/type prototype was manually confirmed before this step.

### Incomplete parenthesized host recovery

An opening host parenthesis now suffices to recognize the Java expression.
Expression ranges count lexer parentheses, ignoring parentheses inside literal
and comment tokens. Missing outer parentheses retain the entire incomplete
content rather than cutting off a complete inner call. Parser recovery stops
at SQLJ block boundaries even with nested unclosed calls, while preserving
balanced Java initializer/lambda braces. Empty and malformed Java fragments
receive syntax ERROR annotations mapped to the SQLJ source.

New tests cover completion with absent host/call closing parentheses, IN mode,
empty/trailing-operator errors and correction. Parser tests check following
statement preservation, inner-call ranges, array braces, string parentheses
and EOF. The previous expression error/completion implementation was manually
confirmed in the installed IDE. This new recovery step still needs local test
execution; Gradle download remains blocked in the author environment.
Next: OUT/INOUT write-target validation, including final variables and array targets.

### Explicit OUT / INOUT write targets

The recovery patch was manually confirmed in the installed IDE. The next step
adds source ERROR annotations for explicit output modes: non-variable results
(method calls, literals, computations), initialized final variables, final
parameters and array length. Writable fields, variables and array elements are
accepted; a final array/receiver reference does not make its contents final.
Malformed and unresolved Java expressions are left to the existing Java checks.
Blank final OUT targets remain undecided until definite-assignment modeling exists.

Thirteen added daemon tests cover acceptance/rejection and recovery after edits.
[Manual scenarios](../../examples/java-view/bindings/README.md) document the
boundary: no implicit SQL-derived modes, procedure metadata, indicator semantics,
Java/SQL conversions or complete data-flow validation yet. The new tests require
execution on the development machine; author-side Gradle download remains blocked.
Next: implicit output targets and indicator-variable semantics in separate steps.

### Implicit SELECT / VALUES / FETCH INTO targets

The explicit write-target patch was confirmed by the user. A conservative PSI
classifier now supplies effective OUT modes for host target lists in top-level
SELECT INTO, VALUES INTO and FETCH INTO clauses. The explicit getMode() API is
unchanged. Java hosts are atomic; comments, quoted text and SQL subqueries do
not switch clause state. INSERT INTO, projection/predicate hosts and the FETCH
iterator are not treated as output targets. Explicit modes take precedence.

Fifteen classification tests and six daemon tests cover boundaries, final/method
outputs, correction, multiple targets and valid final-array/input usage.
The manual fixture is ImplicitWriteTargetProbe.sqlj. This is not a full DB2 SQL
parser: CTEs, compound statements, SET/result assignments and indicators remain
outside inference. No SDK or database dependency changes are introduced.
Tests still need execution on the development machine because author-side
Gradle downloads remain blocked. Next: indicator associations and their modes/types.

### DB2 adjacent indicator pairs

The implicit-INTO patch was accepted by the user. Adjacent `:host :indicator`
pairs now have an explicit ownership relation, inherited effective modes and
primitive-short checks. Existing references supply navigation. Implicit INTO
classification skips indicator nodes so later comma-separated targets still
receive write-target checks. Pairing ignores comments and stops at SQL tokens,
commas and operators; malformed triples do not become recursive chains.

Nine ownership/navigation tests and eight daemon tests cover pairing, direction,
short versus int/Short, final input/output indicators, correction and subsequent
output targets. The FETCH orientation fixture now includes FROM between its
orientation and iterator hosts, avoiding syntactic ambiguity with indicator pairs.
Manual acceptance is in IndicatorProbe.sqlj and the bindings guide. Build execution
is still blocked in the author environment by the Gradle download network error.
Next: expand supported indicator syntax/diagnostics against real customer forms,
then context/result assignment expressions and remaining runtime iterator APIs.

### Complex indicator expressions and excess hosts

While the adjacent-indicator patch is being tested, the follow-up adds IDE-side
handling for parenthesized indicator expressions. Java PSI supplies primitive-short
types, source references and completion; inherited output modes invoke the existing
write-target checker. Short-array elements and fields are supported, including
array contents behind final references. Malformed triples/longer adjacent runs
receive an explicit association error rather than silently creating chains.
Three ownership/reference tests and nine daemon/completion tests extend coverage.

The IBM sources establish short and simple adjacency, but the reviewed pages do
not prove all translator variants of complex indicators or the INDICATOR keyword.
Complex support remains an editor prototype; translator acceptance is a separate
verification gate. The keyword form is intentionally deferred. The author build
cannot run while Gradle distribution downloads report Network is unreachable.
Next: Java context/result assignment expressions, keeping translator checks explicit.

### Java expressions in context qualifiers and result targets

While the indicator patches are tested, the next architecture slice adds Java
references to connection/execution qualifier expressions and result targets
before =. Qualifier parsing retains nested array brackets; slot ranges split
only at outer commas, so ordinary method argument lists remain intact. Result
target parsing accepts this/super, parentheses and indexed targets, and locates
the top-level assignment token. Marked shared-view files create contextual Java
fragments at the native statement anchor. Per-name references map own declarations
to SQLJ source and retain external Java targets. Ten tests cover ranges, lexical
resolution, type access, goto-declaration, external source and missing-bracket
recovery. Manual fixtures are in examples/java-view/headers.

This slice is navigation/type infrastructure only, not full header semantics.
Completion, visible Java errors, connection/execution interface validation and
result write-target checks are next. Fixtures use explicit editor-only placeholders
and do not claim translator-ready queries. Complex generic commas and Java
initializer/lambda braces in headers remain limitations. No SDK change is made;
author-side test execution remains blocked by the Gradle distribution download.

### Header completion, Java errors and result writability

Header ranges now share native Java completion-candidate collection and semantic
highlighting with host expressions. The completion adapter writes to the real
SQLJ document and handles method parentheses. Context/result fragments map Java
ERROR descriptions and syntax ranges back to their owner. Result targets also
reject computations, method returns, initialized final variables/parameters,
compiled/record finals and array length; array elements remain writable. Blank
final targets are deferred as in explicit OUT checks.

Fourteen daemon/completion tests cover errors, disappearance after edits,
member/index completion, insertion, and writable/invalid result targets.
The manual header probe and README are updated. Context runtime-role checks,
SQL return-type compatibility and SQLJ-aware data flow remain next steps;
ordinary Java validity is not advertised as proof of valid SQLJ runtime types.
Author-side execution remains blocked at the Gradle distribution download.

### SQL injection respects indicator pairs

Manual testing found an SQL error mapped to whitespace between a host and complex
indicator. The injector previously generated separate parameter markers for every
host PSI node, yielding INTO ? ? instead of one SQL target. Injection now merges
recognized host/indicator ranges and emits one parameter marker for the pair,
including intervening comments. Java indicator checks/references remain on the
original PSI. Parameter markers are retained even when a host ends immediately
at the SQL block boundary.

Seven new tests reconstruct the exact injected SQL text and check monotonically
ordered source ranges for array/field/simple indicators, mixed pairs/ordinary
hosts, input value lists and trailing hosts. These tests do not replace manual
Database Tools/DB2 annotator validation in the installed IDE. Author Gradle
execution remains blocked by Network is unreachable during download.

### Connection and execution context roles — 2026-10-18

Step 3 now includes static runtime-role checks in marked shared-view SQLJ headers.
A single expression accepts either sqlj.runtime.ConnectionContext or
sqlj.runtime.ExecutionContext; a pair requires connection first, execution second.
The check follows resolved Java inheritance and type-parameter bounds, and
recognizes untranslated #sql context declarations as connection contexts.
Empty slots and more than two expressions receive structural diagnostics.
Unresolved Java references/syntax and missing runtime definitions suppress
secondary role errors. No SQLJ runtime code is shipped with the plugin.

Thirty-two daemon tests cover roles, ordering, JDBC-vs-SQLJ, methods, indexed targets,
inheritance/generics, synthetic contexts, corrections and expression-only ranges.
The earlier header insight fixture now uses explicit runtime-role test doubles;
manual probes use the user's actual runtime on the module classpath. DB2 runtime
behavior, null/default-context availability, role-filtered completion, complete
synthetic runtime APIs and translator-specific expression restrictions remain
separate work. This does not implement debugger source mapping. No SDK upgrade.
Author-side Java tests remain blocked by the Gradle distribution download; run
test/buildPlugin and the installed-plugin scenarios before accepting this slice.

### Synthetic connection contexts inherit the installed runtime API

Untranslated context classes now expose sqlj.runtime.ConnectionContext as an
implemented interface, including Java supertype queries and assignability. Java
method/field lookup and visible signatures use the platform's inheritance helpers.
Inherited runtime methods/constants retain their original PSI, exception lists,
overloads and navigation targets; generated default-context methods still point
to the SQLJ declaration. The runtime is resolved within the SQLJ source's scope,
not deliberately drawn from another module's dependency set. No concrete runtime
implementation superclass is assumed.

With no interface available, the existing lightweight fallback remains and gains
getExecutionContext plus close(). When a real interface is available, it supplies
the instance API instead of being shadowed by fallback methods. Context constructor
coverage remains partial and is not advertised as a complete generated class API.

Twenty tests exercise supertype/inheritance queries, Java assignability and actual
consumer diagnostics, runtime method/constant navigation, close overloads and
exceptions, method signatures, completion/chaining, shared SQLJ headers, missing
runtime fallback and runtime additions/edits. Manual fixtures use the real module
runtime and split Java/SQLJ source roots. Java tests/build remain unexecuted here
because the Gradle distribution download reports Network is unreachable.
Oracle runtime/translator adapters remain deferred until the DB2 integration and
debugger acceptance gates above have passed.

### Context constructor families and with options

The DB2 declaration model now distinguishes URL contexts (five constructors) and
literal-dataSource contexts (four constructors), including both JDBC Connection
and SQLJ ConnectionContext wrapping. All modeled constructors are public, declare
SQLException and retain SQLJ declaration navigation. Constructor families refresh
when the option mode changes, including previously returned model objects.

A shared token-based option model handles comments, string commas, duplicate
attribute names and the DB2 dataSource string literal. Custom Java-ID attributes
are accepted; their Java constant-expression semantics are not evaluated yet.
The with parser balances nested parentheses and stops recovery at statement/new
SQLJ boundaries. Invalid/incomplete options expose only common wrapper constructors.
Manual constructor fixtures use separate Java/SQLJ roots and intentionally bad
calls/options. They are editor fixtures, not JNDI registration or an executable
wizard extension.

The tutorial now distinguishes declared attributes from actual JDBC read-only and
isolation settings. The debugger plan is explicitly container-first: a working
out-of-the-box wizard project is mandatory before alternate/customer toolchains,
remote attach and Oracle adapters. No debugger implementation or wizard-default
behavior is changed by this constructor patch.

### Iterator runtime interfaces and declaration checks

The iterator slice models NamedIterator/PositionedIterator inheritance from the
module's runtime, exposing inherited ResultSetIterator methods/constants and
retaining actual library navigation/exception signatures. Named accessors still
point to their SQLJ columns. Context and iterator classes now share the same
runtime-inheritance implementation, preserving the context behavior from the
previous patch. Offline iterator methods remain a fallback only.

All columns now determine named/positional kind; mixed declarations are neither.
Diagnostics cover mixed columns, case-insensitive duplicate names, empty lists
and void columns. Duplicate accessors are not published twice. Comments are
excluded from type/name extraction; parser recovery retains following SQLJ clauses
and diagnoses trailing commas. Twenty-two tests cover Java assignability, runtime
API navigation/completion, column typing, invalid declarations and fallback behavior.
Manual fixtures live in examples/java-view/iterators, with only their README added
to the public documentation allowlist. Java tests still need the user's build;
local Gradle remains blocked by its distribution download.

Next: explicit iterator implements/with semantics, then the known DB2 container
artifacts for the out-of-the-box wizard debugger. FETCH/result conversion checks
remain pending and must not be inferred from runtime interface support. No alternate
toolchain comparisons or Oracle work are required before that debugger baseline.

### Iterator implements/with: DB2 declaration order and runtime roles

The parser now supports the DB2 prefix order (iterator name, implements, with,
column list). The earlier suffix prototype is retained for editor compatibility;
it is not advertised as container-translator proof. Implemented interface names
have PSI references using Java import/class scope. Resolved explicit interfaces
participate in context/iterator Java inheritance, completion and source navigation.
Unresolved names, non-interface classes and duplicate resolved aliases receive
errors. Generic implemented types, import insertion and rename remain pending.

Context and iterator with clauses share structural parsing, preserving context
constructor mode behavior. Iterator checks cover duplicate/missing attributes,
obviously invalid boolean/sensitivity literals, literal updateColumns and its
ForUpdate requirement, context-only dataSource misuse and known dynamic/SENSITIVE
constraints. Unresolved dependencies defer cascades. dynamic=true warns about DB2
LUW support rather than rejecting every potential z/OS target. Complex constant
expressions/aliases, schema names and actual driver/server effects remain toolchain
checks. No Oracle integration or broader server-profile framework is introduced.

Twenty-seven new tests and manual IteratorOptions/consumer fixtures cover these
paths. The next concrete task is wizard cold-start reliability before debugger
work: the current template runs compose up -d, permits only three minutes of
Spring readiness waiting and has a thirty-second health start period. Inspect and
fix actual database-readiness coordination and initialization budgets, then verify
fresh-container first launch as well as warm launch. This is a source-level finding,
not a reproduced DB2-container failure in the author environment. No startup or
readiness behavior is changed by this iterator patch.

### First debugger slice: runtime coordinates before breakpoint guesses

The existing position manager treated any JVM line as a SQLJ source line, even when
SourceFile still named generated Java. The first slice removes that assumption.
Source lookup now requires matching package/class and exact source name, rejects
ambiguous duplicate qualified sources and out-of-range lines, and defers while
indices are unavailable. JDI source names/lines and breakpoint locations use explicit
available strata; the VM's global default stratum cannot silently change the mapping.
Plain generated Java lines are left to the Java debugger. Conflicting SQLJ strata
are rejected. Source fingerprints are now checked for the generated wizard pipeline described
below; native metadata without fingerprints has no freshness guarantee.

Both wizard translation profiles now request IBM's `-smap=yes`. This emits a sidecar
mapping; with `-compile=false`, subsequent javac does not automatically attach it to
the class. The generated helper described below now installs that sidecar. The fresh wizard's
installed-plugin breakpoint acceptance remains pending. Line-number rewriting with
`-linemap=yes` is not used; Java line tables remain intact.

| Slice | Current state | Acceptance |
| --- | --- | --- |
| Runtime coordinate contract | Implemented; 16 pure-JDI contract tests plus a real child-JVM breakpoint test | SQLJ line 12 maps to generated Java line 5; actual breakpoint hit and local value checked |
| Container translator SMAP output | Requested in both generated profiles; actual container emission pending | Preserve generated `.java`, `.java.smap`, `.class` from one clean wizard build |
| Automatic post-javac mapping | Generated helper implemented; container/IDE acceptance pending | Embed validated container SMAP in matching class files and nested classes; track source/build freshness; preserve ordinary Java debugging |
| Installed-plugin breakpoint acceptance | Pending | Debug the wizard app, hit SQLJ Java/SQL lines and repeated/multiline SQL; unloaded/loaded classes; correct stack source |
| Stepping and evaluation | Existing scaffolding only; pending validation | Step Over/Into, parameters/locals/this, watches/evaluation in the executable Java scope; changed sources fail safely |

The JVM test uses a controlled synthetic JSR-45 fixture and does not require DB2.
It proves the JDI coordinate contract, not IBM translator compatibility or installed
IntelliJ behavior. Additional platform tests cover package/source ambiguity and
package parsing independently of whitespace. Full SDK compilation/tests and a real
container debugger session still require the reference environment.

### Second debugger slice: generated post-javac build helper

The wizard now includes a dependency-free MIT Java build helper. Before translation
it captures source fingerprints and clears stale sidecar maps; after javac it validates
and embeds fresh SMAPs plus source hashes in matching classes, including nested
classes. SourceFile/Java line tables/locals are preserved. The plugin checks hashes
from loaded runtime metadata against current editor text before SQLJ mapping.
The first supported map shape is one stratum and one SQLJ input per map; unsupported
or missing maps fail visibly instead of silently pretending that debugging is enabled.

The real-JVM test uses that generated helper and an external Gradle-toolchain javac,
fixing compiler discovery under the IntelliJ test classloader. Helper tests cover
idempotence, nested/ordinary Java classes, invalid/missing/stale maps, source changes,
old compiled classes, newline differences and paths containing spaces. These tests
still use a synthetic map, not an IBM container artifact.

Next gate: inspect a clean wizard build's actual container SMAP/classes, run the
installed plugin with real SQLJ breakpoints, then wire the Maven preparation into
IDE before-launch behavior. Full Step Over/Into and Java-scope evaluation follow.
Automatic class-loader disambiguation for duplicate qualified sources, native
unfingerprinted toolchains, multi-input/multi-stratum translator formats, customer
artifacts and Oracle remain later work. The first DB2 out-of-the-box debugger goal
is not marked complete by post-compile helper tests alone.

### Wizard integration corrections after the container run

The user's healthcheck log proved that the query succeeded but cleanup in a
separate CLP invocation returned SQL1024N. The probe is now one stop-on-error CLP
batch, and a guarded repair script updates the known old check in existing
starters while retaining a backup. Readiness budgets stay unchanged.

The wizard now registers pom.xml with Maven after project startup, declares source
roots and inherits the project SDK. Maven integration is optional for the plugin;
its bridge uses the bundled 2024.3 MavenProjectsManager API. New starter SQLJ files
enable the shared Java view, and src/main/sqlj survives Maven source-root import.
Class navigation from the known generated Maven directory maps by unique qualified
class name to the original SQLJ identifier in the same content root. Generated
member/source-symbol deduplication and automatic debugger launch preparation are
still pending. Verify a new wizard project's Maven import and class navigation
before and after Maven translation in the installed plugin.

### IDE breakpoint-handler bridge

Installed testing found that source-level breakpoints still did not suspend. Code
inspection identified a missing JavaBreakpointHandlerFactory: IntelliJ dispatches
by exact breakpoint-type class, so extending JavaLineBreakpointType alone left
SQLJ breakpoints without a Java handler. The new bridge registers the exact SQLJ
type with the standard Java request lifecycle. SQLJ has priority for its own files,
uses all-position line properties and checks mapped file/line directly instead of
running Java-only method/lambda PSI checks on SQLJ base PSI.

SDK regression coverage now includes handler extension discovery/type dispatch and
actual native Java LineBreakpoint conversion/source matching. These SDK tests need
the reference build environment; the standalone JDI tests do not exercise this
registration path. Installed breakpoint hits remain the next acceptance gate.
For the manual gate, preserve Maven's mapped output and remove an intervening
Java-only Before launch build. Automatic IDE Maven preparation, full stepping and
evaluation remain pending; no DB2 debugger completion is claimed by this bridge.


### DB2 breakpoint acceptance and reusable launch preparation

Installed-plugin feedback on 2026-10-18 confirms that SQLJ source breakpoints now
hit in the DB2 wizard application after normalizing the observed overlapping DB2
SMAP block ranges. Java debugging and navigation to ProductRepository.sqlj also
work. That acceptance used a manual Maven build with the ordinary before-launch
IDE Build disabled; it does not yet establish automatic launch or full stepping.

The next patch adds the general plugin task **SQLJ: Prepare Debug Build** with
per-run-configuration Maven POM/goals and an optional Maven bridge. The wizard
supplies its defaults through a shared Application run configuration. Older
starters can install that configuration without replacing their existing ones;
existing customer projects can add the task to their own launch configuration.
Their build must supply translation and debug metadata installation. Arbitrary
build/toolchain retrofits are outside this initial Maven orchestration adapter.

Current gate: validate automatic launch, changed-source rebuild and failed-build
launch blocking in the installed plugin. Next: Step Over/Into, stack frames,
locals and expression evaluation. Customer toolchains, remote attach and Oracle
follow the DB2 container baseline; Oracle has not been started.


### Project-type scope and server acceptance

The installed-plugin DB2 wizard's SQLJ breakpoints and automatic Maven before-launch
preparation are now user-confirmed. Spring Boot is only that acceptance fixture,
not a dependency of the Java/JDI debugger integration. Plain Java, Spring without
Boot and Spring hosted by WebSphere Liberty (WLP) remain explicit targets.

[The project-type debugging guide](../user-guide/debugging-project-types.md)
records IDE/Maven prerequisites and the build/package/deploy/attach boundary.
Record a dedicated Liberty acceptance run before claiming it is validated:
Java and SQLJ breakpoints, source edits plus rebuild/redeployment, nested iterator
classes, source freshness, duplicate class names/class loaders and stop/reattach.
The initial server run uses a controlled packaged artifact; dev-mode recompilation
and HotSwap are separate follow-up integration work. A generic before-launch Maven
build does not by itself deploy to or update an already running server.

This adds server setup/acceptance scope without starting Oracle integration or
requiring a Boot dependency in general plugin functionality.


### Stepping policy and variable/frame regression

The next patch scopes the extra stepping filter to SQLJ/DB2 runtime locations and
uses STEP_OUT there. The previous always-applicable STEP_INTO behavior is removed
so unrelated Java, Spring and Liberty code retains native stepping behavior.
A controlled real-JVM fixture covers mapped Step Over/Into/Out and locals/caller
frames. [Manual acceptance](../../examples/debugging/README.md) remains required
for IntelliJ commands against the actual DB2 application. This does not complete
Watches/Evaluate Expression or the separate Liberty deployment/attach acceptance.


### Watches and Evaluate Expression: lexical preparation

The next slice preserves the suspended SQLJ method/block context when preparing
native Java debugger expressions. It adds current SDK evaluation/presentation
hooks and compatibility bridges, and removes class-level context replacement.
Thirteen SDK scenarios cover lexical scope, shadowing, this, imports, nested/static
contexts and fragment entry points. Manual Watches/Evaluate acceptance is now
specified in [the debugger scenario](../../examples/debugging/README.md).

Remaining gate: record actual Watches, Alt+F8 evaluation and frame switching in
the installed DB2 wizard application, followed by conditions/log expressions.
PSI resolution and JDI local reads are distinct evidence. Liberty deployment/attach,
full customer toolchain compatibility and Oracle remain separate later work.


### Native conditional/log breakpoint preparation

A Java-file-type alias of the SQLJ fragment factory closes the explicit-language
dispatch gap for Watches, conditions and log expressions. Context acceptance stays
restricted to SQLJ source providers, including native Java sibling roots. Existing
SQLJ-labelled expressions keep the original registration. Native Java condition
execution, log actions and suspend policies remain unchanged.

Ten SDK scenarios exercise dispatch, lexical preparation and native SQLJ breakpoint
settings. [Manual acceptance](../../examples/debugging/README.md) now includes a
three-invocation conditional hit and non-suspending logs with local/field shadowing.
Actual IDE execution, expression errors and combined condition/log behavior remain
the current gate. Dedicated Liberty deployment/attach acceptance follows; no
Spring Boot requirement or Oracle integration has been introduced.

### Debugger diagnostics checkpoint

The paused-session inspection action now reports loaded SMAP/provenance,
class-loader identity, editor-line locations and expression context for both
local Java sessions and remote Java sessions such as Liberty. It does not
perform deployment or infer artifact identity from a matching source hash.
Next acceptance gate: exercise the container-generated DB2 project with
breakpoints, stepping, locals, watches, conditional and log breakpoints together.
Then validate deployment/attach and source matching in a real Spring/Liberty
project. Oracle SQLJ remains deferred until the DB2 integration/debugging
baseline is accepted.
