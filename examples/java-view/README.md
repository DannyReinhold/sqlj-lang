<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Experimental shared-document Java view

This is an architecture probe, not a supported production editing mode.
Only files whose first characters are `// sqlj:experimental-java-view` use it.
Other files keep the previous view. Adding/removing the marker requires reopening
the project so IntelliJ recreates the file view provider. Start with a copy in a
scratch project with a configured JDK; do not opt customer sources in yet.

Open `JavaViewProbe.sqlj` in that project. Invoke completion after `connection.`,
`names.get(0).` and `first.`. Expect native Java proposals, type substitution,
parameter-aware insertion and navigation. Try a partial member name as well.
Check imported types, method parameters, and Java navigation across the SQL block.
Temporarily change the SQL to `SELECT FROM` while retaining `};`; Java assistance
after it should survive. Change a Java method or parameter and repeat the checks.

## What the probe establishes

- One document/virtual file with SQLJ and Java PSI roots, rather than a detached
  `.projected.java` file for each lookup.
- SQLJ lexer-based template splitting, preserving strings/comments and original
  offsets through IntelliJ's template-data infrastructure.
- Native Java completion should receive Java PSI in Java regions. The new tests
  explicitly expect Java lookup names, not the old handcrafted `method()` names.
- The scope resolver reuses the Java root for opted-in files.

## Open gates and known limitations

Tests have been added but cannot be run in the author's current execution environment.
Passing the packaged IDE checks is required before enabling this for all files.
The SDK remains IC 2024.3.4; no platform version is changed by this prototype.

This is not a new SQLJ semantic model: host expressions and iterator/context
semantics remain incomplete. SQLJ statements are hidden from Java parsing, so
Java definite-assignment/exception inspections can report false positives for SQL
output bindings or SQL-thrown exceptions. SQLJ expressions used as Java values,
unterminated SQL blocks, and all translator-specific forms need further design.
A malformed SQL statement WITH a closed SQLJ block is a different test from a
missing closing brace; recovery for the latter is not claimed.

Before adopting this architecture, verify Java-to-SQLJ class lookup, synthetic
iterator identity, annotations, inspections, imports, editing/formatting and
language routing inside SQL/host expressions. Generated Java must not introduce
duplicate symbols. Source debugging still needs independent artifact mapping.

## Iterator, context and Java consumer probe

Copy `interop/Repository.sqlj` and `interop/JavaConsumer.java` into a scratch module
with a configured JDK (and SQLJ runtime for real translation). Put the Java source
under `src/main/java/examples/interop/client/JavaConsumer.java`. Keep the SQLJ file
in the same module, e.g. `src/main/sqlj/examples/interop/Repository.sqlj`.
Do not run translation for the first checks; generated classes would obscure
whether source-only lookup works.

- In both files, complete members on `rows` and `context`. Expect each receiver's
  own methods, with Java parameter information and normal Java insertion.
- In the Java consumer, navigate from `employeeId()` to the SQLJ column and from
  `identity()` to the method body in the original SQLJ file.
- Complete after `repository.identity("example").`: expect String members.
- Rename a column manually in the SQLJ declaration, then inspect proposals again.
  Cached source classes should update after the document is committed.
- Check imported iterator/context types resolve even before a generated Java root
  or a matching physical package directory exists.

This patch covers top-level declarations. Nested iterator/context ownership,
translator-complete runtime APIs, declaration visibility, generated-source duplicate
handling and cross-language Rename/Find Usages remain separate acceptance gates.
The opt-in marker is still required for the shared Java view.

### Editor navigation regression check

Use Ctrl+B / Go to Declaration on the method NAME (not only on the receiver type)
in `JavaConsumer.java`, with no generated Java present. `employeeId()` must target
the `employeeId` column name, and `identity()` the method name in `Repository.sqlj`.
Repeat `employeeId()` from the experimental SQLJ Java view. The navigation handler
now maps resolved Java/light targets to concrete leaves in the original SQLJ root.
`SqljJavaGotoDeclarationTest` checks the editor's declaration-target pipeline;
`resolveMethod()` alone is insufficient evidence for this behavior.


### Navigatable source targets

The SQLJ navigation handler now returns an explicit source destination with an
OpenFileDescriptor, rather than relying on a raw SQLJ token's default navigation.
The regression suite additionally calls the actual Ctrl+B pipeline's outcome
helper (which builds a navigation request), and checks that the explicit target
opens the original file at the declaration name. A successful old
`GotoDeclarationAction.findTargetElement()` test alone is not sufficient.

## Nested declaration probe

Use `nested/NestedRepository.sqlj` and `nested/NestedConsumer.java` together in a
module with a JDK. For the initial check keep their declared packages AND imports
unchanged: `examples.nested` / `examples.nested.client`. If you move the example
into another package, update every import and qualified reference accordingly.
The SQLJ file must retain the experimental marker. No translation is required.

- Complete on `rows.`, `context.` and `departmentRows.` from both applicable files.
  Each receiver must expose its own API; the two EmployeeRows types are distinct.
- In a Java type position, complete `NestedRepository.`. Public nested types should
  appear; private InternalRows must not be offered to the external consumer.
- Navigate from employeeId() to its SQLJ column and from AppContext to its SQLJ
  declaration. Repeat after opening, editing and saving the SQLJ file.
- Check that `examples.nested.EmployeeRows` is not accepted as a top-level type;
  the correct name is `examples.nested.NestedRepository.EmployeeRows`.

The plugin now models member ownership through Java PSI augmentation, including
multiple containing classes and declared visibility/static modifiers. Interface
member types get Java's implicit public/static visibility. Method-local SQLJ
declarations are deliberately not exported as member/package types; local type
semantics, complete translator APIs and generated-source deduplication are pending.
The change is scoped to files with the shared Java view; legacy ownership is not
upgraded by this patch.

## Java error analysis

The [highlighting probes](highlighting/README.md) cover bad imports, unknown types,
unknown methods and invalid argument lists in .java and the shared .sqlj Java view.
They also cover correcting each error without reopening the file. This is a
separate acceptance gate from completion/navigation and must be checked in the
installed plugin, not just by resolving PSI references in tests.

## Host binding scopes

The [scope probe](scopes/README.md) checks block boundaries, shadowing, declaration
order and static context, plus ordinary names beginning with in/out. Host references
and the custom completion contributor now use Java's lexical scope traversal.
Inherited fields retain their original Java navigation targets.

## Context header types

The [header probes](headers/README.md) now cover runtime roles: a single
ConnectionContext or ExecutionContext, ordered connection/execution pairs,
untranslated SQLJ contexts, Java factories, array elements and invalid static
types. Put the SQLJ runtime on the module classpath and keep the shared-view marker.

The same header guide now includes ContextApiProbe.sqlj and a separate ordinary
Java consumer. They check runtime-interface assignability, inherited methods and
constants, checked exceptions, chained ExecutionContext completion and navigation
to the actual runtime library before any SQLJ translation.

## Iterator runtime interfaces

The [iterator API probes](iterators/README.md) cover named/positional Java interface
assignability, generated getters, inherited runtime methods/constants, JDBC chaining,
source navigation and invalid mixed/duplicate/void column declarations.
