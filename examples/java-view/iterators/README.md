<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Iterator runtime APIs and declaration checks

Copy IteratorApiProbe.sqlj to src/main/sqlj/examples/iterators and
IteratorApiConsumer.java to src/main/java/examples/iterators in a manual test
project. Keep the package names and first-line SQLJ marker, and put the actual
DB2 SQLJ runtime on the module classpath. No translation or database is required.
These fixtures intentionally contain errors and are not a runnable wizard example.

| Check | Expected behavior |
| --- | --- |
| NamedRows assigned to NamedIterator or ResultSetIterator | no Java type error |
| PositionRows assigned to PositionedIterator or ResultSetIterator | no Java type error |
| NamedRows assigned to PositionedIterator | Java type error |
| `named.` completion | employeeId/label/payload plus actual inherited runtime methods |
| `positioned.` completion | endFetch and runtime methods, no named column getters |
| `named.getResultSet().` completion | JDBC ResultSet API |
| `named.endFetch()` / `positioned.label()` | Java errors with the standard runtime |
| Ctrl+B on a named getter | its SQLJ column |
| Ctrl+B on next/close/getResultSet/endFetch or inherited constants | actual runtime source/decompiled library declaration |
| MixedRows | error: all columns must be named or all positional |
| DuplicateRows | error: column names are unique ignoring case |
| InvalidRows | error: a column cannot have type void |

Fix the bad declarations/calls and check that diagnostics disappear without
reopening. Remove a closing iterator parenthesis before a following #sql clause:
that following clause must remain parsed. A trailing column comma must produce
a parser error. Comments within a column declaration must not corrupt the Java
type. Duplicate getter names are not emitted twice; mixed declarations do not
publish a misleading named API.

With the real runtime available, named/positioned synthetic classes implement
sqlj.runtime.NamedIterator/sqlj.runtime.PositionedIterator and inherit the library's
ResultSetIterator methods, constants and exceptions. Generated column getters
retain SQLJ navigation and declared types. Without those interfaces a small offline
fallback remains; it does not install fake runtime classes or prove assignability.
Context and iterator inheritance use the same platform-based implementation.

This slice does not validate FETCH column counts/conversions, iterator-assignment
query results, SQL schema aliases, generic implemented types, complete with-value expression semantics, generic column syntax, direct iterator construction or translator-specific internal methods.
Those require later slices. No Oracle integration or debugger change is included.
The first debugger target remains the fresh wizard project and DB2 container only.

References: [IBM iterator declarations](https://www.ibm.com/docs/en/db2/11.1?topic=information-sqlj-iterator-declaration-clause)
and [iterator names/column rules](https://help.sap.com/saphelp_SCM700_ehp02/helpdata/en/eb/4a21346fb369429a1fce29cfb29f16/content.htm).

## implements / with declarations

Copy IteratorOptions.sqlj into src/main/sqlj/examples/iterators and
IteratorOptionsConsumer.java into src/main/java/examples/iterators. Keep the real
DB2 runtime on the module classpath. These are editor-only fixtures with deliberate
errors, not files to include in the runnable wizard example.

DB2 places implements and with before the column list. The parser now handles:

```java
#sql public iterator ScrollRows implements sqlj.runtime.Scrollable, sqlj.runtime.ForUpdate
    with (holdability=true,
          sensitivity=sqlj.runtime.ResultSetIterator.INSENSITIVE,
          updateColumns="employeeId") (int employeeId, String label);
```

The previous prototype's suffix form is retained for editing compatibility only;
use the DB2 prefix form for translator acceptance. Do not infer that every parsed
editor form has been validated by the DB2 container translator.

- Assign ScrollRows/ScrollPositionRows to their normal iterator role and Scrollable;
  assign ScrollRows to ForUpdate. These Java statements must have no type error.
- Complete after named./positioned. in the consumer: the actual Scrollable API
  (first, absolute, etc.) should appear alongside the iterator's own column API.
- Ctrl+B on Scrollable/ForUpdate in the SQLJ declaration should reach the actual
  Java interface (including when the name is imported). Runtime methods retain
  runtime navigation; generated column methods retain SQLJ-column navigation.
- MissingUpdateRole must report that updateColumns requires ForUpdate.
  BadHoldability and DuplicateOption must report their literal/duplicate errors.
  WrongInterface must report that java.lang.String is a class, not an interface.
- DynamicServerOnly has a DB2 LUW compatibility WARNING, not a universal type error:
  supported z/OS server combinations can differ. Explicit INSENSITIVE/ASENSITIVE
  or default sensitivity with dynamic=true is rejected; dynamic=true also needs
  a Scrollable interface.
- Fix bad values/interfaces, verify the errors disappear, then remove a comma or
  interface name and confirm following declarations remain parsed.

Interface names resolve in the projected Java import/class scope, and resolved
interfaces participate in Java inheritance. Duplicate aliases are diagnosed and
are not added twice. Unresolved names are explicit interface errors, but dependent
ForUpdate/Scrollable diagnostics are deferred when an unresolved interface might
supply that role. The same reference mechanism is shared by context implements
clauses. Generic implements syntax, automatic imports and rename remain pending.

with parsing is shared with context options and respects comments, literal commas
and nested parentheses. Checks cover structural/duplicate attributes, clearly wrong
boolean literals, a string-literal updateColumns value and its ForUpdate dependency,
clearly wrong sensitivity literals, the misplaced context-only dataSource option
and the dynamic/SENSITIVE relationship. Arbitrary Java constant expressions and
aliases are not evaluated or fully type-checked here; unknown custom attributes
remain allowed. This is not a schema/column existence check or a proof of DB2 server
support. Driver customization, holdability effects and actual scroll/update behavior
still require the container/toolchain acceptance run.

Reference: [IBM DB2 scrollable iterator sample](https://github.com/IBM/db2-samples/blob/master/java/sqlj/ScrollIterDemo.sqlj)
and [DB2 with-clause rules](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=ssri-sqlj-clause).
