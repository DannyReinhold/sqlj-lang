<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Parenthesized Java host expressions: resolution and navigation

Copy ExpressionProbe.sqlj to src/main/sqlj/examples/expressions in the test
project. Retain the experimental marker. Ordinary Java helpers belong under
src/main/java with matching packages. No translation or database is needed.

Use Ctrl+B / Go to Declaration on each name inside the host expression:

| Expression | Expected navigation |
| --- | --- |
| :( employee.getId() ) | employee -> method parameter; getId -> nested Employee method |
| :(values[index + 1]) | values -> parameter; index -> local variable, not field |
| :IN (identity(identity(index))) | both identity names -> method; index -> local |
| :(employee.doesNotExist()) | employee resolves; doesNotExist has no target |
| :(ended + 1) | ended has no target, because its block has ended |

Repeat after saving and reopening the file. Whitespace and nested parentheses
must not shift the navigation range. Test an external Java helper under
src/main/java as well: its method must navigate to its real .java source.

This step parses contextual Java expression fragments, exposes individual
references and derives their expression types through IntelliJ's Java PSI.
Automated tests cover primitive result types, lexical shadowing, array indices,
unknown members, expired scope, external source and explicit IN mode.

Member completion and native Java ERROR checks are now adapted to expression
fragments. The negative probes should turn red; correcting doesNotExist to getId
and removing the extra argument must remove those errors without reopening.
The expired name ended must be reported as unresolved.

For completion, temporarily replace employee.getId() with employee.g and invoke
Ctrl+Space before the closing host parenthesis. getId must be offered without SQL
keywords. Selecting it inserts getId(); an existing argument list must not be
duplicated. Try employee. with an empty prefix, (employee).g for a parenthesized
receiver and ind for a visible local name. Restore the expression afterwards.

Java's semantic visitor supplies messages, including overload/argument errors;
only ERROR severity and source ranges are copied. Fragment quick fixes and
imports are deliberately not copied into the SQLJ document. During indexing,
these checks and candidates are deferred. This is not complete SQLJ-aware
flow analysis or bind-direction validation.

## Incomplete expressions while editing

Starting from the working first query, remove its host closing parenthesis,
then replace the call with `employee.g`. Invoke Ctrl+Space before the SQLJ `}`:

```sqlj
#sql { VALUES :(employee.g };
#sql { VALUES :(employee.withIndex(employee.g };
#sql { VALUES :IN (employee.g };
```

The first and third forms must offer getId. For the nested form, use an employee
method accepting an int (or add `int withIndex(int value) { return value; }` to
Employee); getId completion must still work for the inner receiver. The missing
parentheses must produce a syntax error, while the following SQLJ query remains
independently parsed and navigable. Restore all required parentheses and verify
the syntax error disappears without reopening the file.

Also try `:()` and `:(index +)`: both must show a Java syntax error. Correct them
to `:(index)` and `:(index + 1)` and confirm the errors disappear. Parentheses in
string literals and Java array initializer braces must not terminate the host.

Recovery recognizes the SQLJ closing brace, a top-level semicolon and end of
file. It cannot reliably distinguish a SQLJ closing brace from a missing Java
lambda/initializer brace in every malformed expression; that ambiguity remains.

Remaining limits: rename, auto-import, parameter-info UI, OUT/INOUT assignability,
indicator associations and Java evaluation/debugging.
Legacy SQLJ files without the shared-view marker are not covered by this feature.
