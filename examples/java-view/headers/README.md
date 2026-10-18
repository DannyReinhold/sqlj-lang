<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Java expressions before the SQLJ code block: navigation prototype

Copy HeaderExpressionProbe.sqlj to src/main/sqlj/examples/headers in the manual
project and retain the shared-view marker. No translation or DB2 connection is
required. Context and Object here are **editor-only placeholders**, not actual
SQLJ connection/execution types. VALUES plus a result assignment is likewise a
placeholder for exercising Java navigation, not a translator-ready SQLJ clause.

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

The context examples use placeholders: Java validity does not prove that the
objects implement the required SQLJ runtime roles. Connection/execution interface
validation, SQL result type compatibility, data-flow effects, rename, auto-import
and debugger integration remain separate work. Legacy files without the marker
do not use these Java fragment checks.

Additional acceptance:
- Fix unknownContext to ctx and unknownResult to result; both Java errors disappear.
- Change doesNotExist() to getConnection() and choose(1) to choose(1, 2).
- Replace the invalid result targets with result or results[index].
- At `[ctx.g<caret>]`, Ctrl+Space must offer getConnection/getOther and selecting
  getConnection must insert one pair of parentheses.
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
