<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Integration roadmap — 2026-10-17

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
- Customer builds own translation/binding. Debugging must attach to their artifacts;
  never silently rebuild them using the development container's translator.

## Delivery steps and acceptance gates

| Step | Deliverable | Acceptance gate |
| --- | --- | --- |
| 1 — started | Manual examples, honest coverage matrix, receiver-based member completion | Two iterator types do not leak APIs; ordinary Java receiver uses its own type; packaged IDE check still required |
| 2 | Java editor architecture spike: shared-document Java view/projection versus Java-oriented PSI | Imports, overloads, generics, chained calls and Java-to-SQLJ API lookup work before translation; incomplete SQL preserves surrounding Java; choose architecture from evidence |
| 3 | Shared semantic model for host bindings and SQLJ declarations | Correct lexical scope/shadowing, bind expressions, contexts, named/positional iterators, indicators/index forms; navigation and completion use identical symbols |
| 4 | Artifact-driven debugger mapping | Real JDI breakpoint hit, SQLJ source location, Step Over, locals and evaluation for supported translator output |
| 5 | DB2 database insight | Selected dialect/data source/schema, aliases, tables/columns, functions, offline behavior and SQLJ-aware SQL projection |
| 6 | Cross-language references and refactoring | Find Usages/Rename across Java and SQLJ; no generated-file edits or duplicate symbols |
| 7 | Toolchain and release hardening | PATH detection, multi-module roots, stale outputs, translator diagnostics, compatibility and packaged-plugin verification |

Step 1 is a bounded correction on the existing projection, not the architecture
choice for step 2. It does not yet promise complete Java accessibility rules,
static/instance filtering, overload-aware insertion or Java inspections.

## Debugger portability

Collect a minimal SQLJ source, matching generated Java and class files, build
options, translator/JDK versions and `javap -v -l` output for each supported setup.
Use synthetic/anonymized reproductions for customer toolchains.
Inspect SourceFile, LineNumberTable, LocalVariableTable and SourceDebugExtension;
check JDI available strata and source names before choosing a mapping strategy.

Prefer verified translator mappings. Use an adapter for generated source metadata
only when it is demonstrably reliable. With no reliable mapping, report that
limitation instead of assuming identical SQLJ/Java line numbers. Detect stale or
mismatched artifacts. Local and remote attach must use the same mapping model.
A database bind step and a translator's debug metadata are separate concerns.

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
