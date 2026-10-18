<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Java expressions before the SQLJ code block: navigation prototype

Copy HeaderExpressionProbe.sqlj to src/main/sqlj/examples/headers in the manual
project and retain the shared-view marker. No translation or DB2 connection is
required. Add the customer's SQLJ runtime (for example sqlj4.zip) as a module library
so sqlj.runtime.ConnectionContext and sqlj.runtime.ExecutionContext resolve in the
IDE. The abstract nested Context implements the real connection interface without
claiming a complete concrete implementation. VALUES plus a result assignment is
still an editor-only fixture, not a translator-ready SQLJ clause.

Use Ctrl+B / Go to Declaration:

| Source | Expected target |
| --- | --- |
| ctx in the first qualifier | inspect parameter |
| execution in each qualifier | inspect parameter |
| contexts and index inside the indexed qualifier | respective inspect parameters |
| choose in the third qualifier | nested Context.choose method |
| result in this.result | the instance field |
| results and index before = | field and parameter, independently |
| unknownContext / unknownResult | no target |

The comma inside choose(1, 2) must not create an additional context slot. An
indexed context must retain the outer closing ] and the result target that follows.
Repeat after edits, saving and reopening. An external Java context factory method
should navigate to its .java declaration under src/main/java.

Recovery check: remove only the final outer ] from [contexts[index], execution].
A parser error must appear, while the SQL block and the following SQLJ statement
remain parsed. Restore ] and verify navigation again.

This step now adds header member/local completion, Java syntax/semantic ERROR
annotations and result-target writability checks on top of navigation. Unknown
names/methods, wrong call arguments, initialized final result variables, method
results and array length receive errors. Writable array elements remain valid,
including elements behind final array references. Blank final result targets
remain deferred until definite-assignment modeling exists.

Context runtime roles are now checked as described below. SQL result type
compatibility, data-flow effects, rename, auto-import and debugger integration
remain separate work. Legacy files without the marker do not use these Java
fragment checks.

Additional acceptance:
- Fix unknownContext to ctx and unknownResult to result; both Java errors disappear.
- Change doesNotExist() to getContext() and choose(1) to choose(1, 2).
- Replace the invalid result targets with result or results[index].
- At `[ctx.g<caret>]`, Ctrl+Space must offer getContext/getOther and selecting
  getContext must insert one pair of parentheses.
- At `results[ind<caret>] =`, Ctrl+Space must offer index (a unique candidate may
  be inserted immediately). Member completion before = uses the result receiver.
- Remove a call parenthesis inside a context slot, observe its Java syntax error,
  restore it and confirm the error disappears without reopening.

The splitter handles two ordinary context expressions, nested Java parentheses
and array indices. Explicit generic method type arguments containing commas,
Java lambda/array-initializer braces, unusual translator-specific header forms
and full recovery for malformed Java expressions remain outside this prototype.

Reference syntax examples:
- [IBM SQLJ iterator assignment](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=applications-using-positioned-iterator-in-sqlj-application)
- [SQLJ context/result expressions](https://docs.oracle.com/en/database/oracle/oracle-database/19/jsqlj/basic-language-features.html)

## ConnectionContext / ExecutionContext type probe

Copy ContextTypeProbe.sqlj beside the other probe, keeping package examples.headers
and its first-line marker. The real SQLJ runtime must be on the module classpath;
no IBM runtime sources or binaries are bundled by the plugin. No SQLJ translation
or database connection is needed for editor checks.

| Case | Expected role check |
| --- | --- |
| `[connection]`, `[execution]`, `[declared]` | valid: a single slot accepts either role, including an untranslated SQLJ context |
| `[connection, execution]`, `[declared, execution]` | valid: connection first, execution second |
| `[connections[index], execution]`, `[connection(), execution()]` | valid: use the Java expression's static type |
| `[execution, connection]` | ERROR on both expressions |
| `[connection, connection]` | ERROR on the second expression |
| `[opaque]`, `[index]`, `[jdbc]`, `[connections]` | ERROR: Object, int, JDBC Connection and an entire array are not SQLJ contexts |
| `[connection, opaque]` | ERROR: execution slot requires ExecutionContext |
| `[connection, execution, execution]` | ERROR: at most two context expressions |
| `[, execution]`, `[connection, ]`, `[]` | ERROR: missing context expression |
| `[missing]`, `[connection.missing()]` | Java ERROR, without an additional role error |

Change `[opaque]` to `[execution]`: the role error must disappear without reopening.
Repeat with an implementation/subclass from the customer runtime and a factory
returning that type. Inherited interfaces, subclasses and generic upper bounds
participate in the static role check; a variable declared Object is not accepted
merely because its runtime value might be a context. Missing runtime definitions
or unresolved Java type hierarchies defer the corresponding role diagnostics.
Nullness, default-context availability and translator-specific restrictions on
complex context expressions are not checked. Completion still offers ordinary
Java members; it is not filtered to context-compatible results.

Sources: [IBM context clause](https://www.ibm.com/docs/en/db2/11.1.0?topic=information-sqlj-context-clause)
and [SQLJ single/paired context forms](https://docs.oracle.com/en/database/oracle/oracle-database/19/jsqlj/basic-language-features.html).

## Synthetic context runtime API

Copy ContextApiProbe.sqlj to src/main/sqlj/examples/headers and
ContextApiConsumer.java to src/main/java/examples/headers in the same manual
project. Preserve the package and first-line marker. Put the actual DB2 SQLJ
runtime on the module classpath. Do not add fake sqlj.runtime classes to a real
project. These are editor fixtures: no translation or database is needed, and
the close calls must not be run against a production connection.

1. Neither file should report a Java type error on `ConnectionContext view = context`
   (or `asRuntime = context`) or `accept(context)`.
2. Complete after `context.`: getConnection, getExecutionContext, close and isClosed
   should come from the runtime API available in this module.
3. Complete after `context.getExecutionContext().`: the actual ExecutionContext
   API should appear (for example getUpdateCount).
4. Complete after `ApiContext.`: getDefaultContext/setDefaultContext remain generated
   SQLJ members; KEEP_CONNECTION/CLOSE_CONNECTION come from the runtime interface.
5. Ctrl+B on getExecutionContext/getConnection/close or a runtime constant should
   open the real runtime declaration (or its decompiled library declaration), not
   a fabricated SQLJ method. Navigate on ApiContext or its default-context methods
   to reach the SQLJ declaration instead.
6. Both close overloads must resolve, with the runtime's checked exceptions. Remove
   `throws SQLException` from inspect and verify that Java reports the unhandled
   close exception; restore it afterward.
7. Temporarily assign `ExecutionContext wrong = context`: expect a Java error.
   Restore the valid code and check that the error disappears without reopening.
8. Verify the `[context, context.getExecutionContext()]` qualifier has no role error.

The installed module's runtime interface and its superinterfaces are the source of
inherited methods, overloads, constants and navigation targets. Runtime changes
invalidate PSI caches. The plugin does not model the context as a DefaultContext
subclass or invent a translator-specific implementation base class. Without the
runtime, only generated default-context methods and a small instance-method
fallback remain; missing library types are not silently replaced with fake runtime
classes. The constructor model is extended by the following slice. Additional generated
profile methods, custom attribute constant-expression evaluation, declaration
implements clauses and other translator variants remain separate work.

Reference: [IBM ConnectionContext API](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=package-sqljruntimeconnectioncontext-interface).

## Context constructors and dataSource options

Copy ContextConstructors.sqlj under src/main/sqlj/examples/headers and
ContextConstructorConsumer.java under src/main/java/examples/headers. These are
manual editor fixtures with intentional errors; do not add them to the runnable
wizard project build. No DB2 connection or translation is needed for these checks.

| Declaration | Supported constructor signatures |
| --- | --- |
| URL context (without dataSource) | `(Connection)`, `(ConnectionContext)`, `(String, boolean)`, `(String, String, String, boolean)`, `(String, Properties, boolean)` |
| `with (dataSource = "jdbc/sqlj-demo")` | `()`, `(String, String)`, `(Connection)`, `(ConnectionContext)` |

All constructors are public, declare SQLException, and navigate to their SQLJ
context declaration. Complete or inspect parameter info after `new UrlContext(`
and `new DataSourceContext(`; the sets must differ. Both wrapper constructors
are present, so untyped `null` is ambiguous. An explicit JDBC/SQLJ cast selects an
overload but does not prove the runtime connection is usable.

The three deliberately bad option declarations should highlight the duplicate
attribute, non-string dataSource value and missing value. Fix them and check that
the errors disappear. Removing dataSource (or renaming it to a user-defined `tag`)
changes the DataSource constructor set back to the URL set without reopening.
Missing parentheses are parser errors; malformed options conservatively expose
only the common wrapper constructors, not a fabricated union of both APIs.

Names and values are read as tokens, not substring matches. Comments and strings
containing commas/dataSource do not create attributes; nested custom expression
parentheses preserve following options/declarations. User-defined Java-ID
attributes are allowed, with duplicate-name checks, but their Java constant
expressions are not yet type-checked or evaluated. The DB2 dataSource form here
requires a string literal. Logical JNDI registration, credentials, availability,
readOnly/isolation effects and actual DB2 translator output remain runtime/toolchain
checks. In particular, declaring a dataSource does not register a JNDI source.

The out-of-the-box debugging baseline remains the runnable wizard-generated DB2
container project; it must not require these JNDI fixtures or a customer toolchain.
