<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# SQLJ forms and editor checks

This is a practical guide to the forms modeled by the plugin, **not a complete
SQLJ/DB2 language specification**. The [capability matrix](compatibility.md)
describes prerequisites and limits. Your IBM translator/runtime defines which
programs execute successfully. Examples below are fragments for a configured
SQLJ project, not complete standalone builds.

## Statements and iterator assignment

```java
#sql { UPDATE employee SET name = :name WHERE id = :employeeId };
#sql [connectionContext] rows = { SELECT id, name FROM employee };
```

An executable clause ends with a semicolon. SQL is enclosed in braces; a qualifier
selects connection/execution context, and an assignment target receives an iterator.
The IDE recognizes these headers and Java targets. It does not independently
validate every SQL statement or SELECT result against the database schema.

## Named and positional iterators

```java
#sql public static iterator EmployeeRows(int id, String name);
#sql iterator EmployeeValues(int, String);
```

Named iterators expose declared accessors such as `rows.id()` and `rows.name()`
after `rows.next()`. Positional iterators use `FETCH` output binds and `endFetch()`:

```java
int id = 0;
String name = null;
#sql { FETCH :values INTO :id, :name };
if (!values.endFetch()) {
    System.out.println(id + ": " + name);
}
```

Close iterators in `finally` according to the runtime API. Declaration and modeled
runtime members support completion/navigation before translation; this does not
verify SELECT-column compatibility. See the [iterator tutorial](../tutorials/working-with-iterators.md).

## Host values, output targets and indicators

| Form | Meaning / editor behavior |
| --- | --- |
| `:value` | Java host reference; direction also depends on SQLJ position, e.g. output in `SELECT INTO`. |
| `:IN value` | Explicit input. |
| `:OUT value` | Explicit output; must be a writable target. |
| `:INOUT value` | Read/write; must be a writable target. |
| `:(employee.getId())` | Parenthesized Java input expression; Java member/type insight. |
| `:(values[index])` | Java array access, including resolution of the index expression. Output positions also require writability. |
| `:value :indicator` | One value bind plus its indicator, not two result columns. |
| `:value :(indicators[index])` | Parenthesized indicator expression; primitive `short` required. |

Input expressions are subject to translator/runtime type support; the IDE does
not guarantee that every Java type/expression is a valid SQLJ host value.
Output binds cannot write into method-call results or final variables. Primitive
`short` indicators can be variables, fields or array elements. Input indicators
can be short-valued expressions; output/INOUT indicators must also be writable.

```java
int value = 0;
short[] indicators = new short[1];
int index = 0;
#sql { SELECT 1 INTO :value :(indicators[index]) FROM SYSIBM.SYSDUMMY1 };
```

Indicator values represent NULL/non-NULL and DB2-specific statuses according to
the translator/runtime; see [IBM's indicator documentation](https://www.ibm.com/docs/en/db2/11.5.x?topic=programming-indicator-variables-in-sqlj-applications).
The plugin checks Java types/targets, not all database-level indicator semantics.

## Connection and execution contexts

```java
#sql public context AppContext;
#sql public context ReportingContext with (dataSource = "jdbc/reporting");

#sql [connectionContext] { UPDATE employee SET active = 1 };
#sql [executionContext] { UPDATE employee SET active = 1 };
#sql [connectionContext, executionContext] { UPDATE employee SET active = 1 };
```

Qualifier entries are Java expressions. One entry may have a ConnectionContext or
ExecutionContext role; two entries are connection then execution. The editor
checks these roles when runtime types resolve and accepts modeled context
classes before translation. Syntax/type checks do not create a connection.

A modeled context without `dataSource` offers wrappers for an existing JDBC
Connection or SQLJ ConnectionContext, plus URL/autocommit, URL/user/password/
autocommit and URL/Properties/autocommit constructors. With a literal `dataSource`,
the modeled constructors instead include the wrappers, parameterless and
user/password forms. They declare SQLException. Verify the actual generated API
against the deployed IBM runtime.

The `dataSource` name refers to an existing JNDI resource. The plugin does not
provision it. Duplicate options and selected known option values are checked;
other user-defined `with` attributes do not automatically become JDBC properties.
See [connection/context setup](../tutorials/db2-connection-and-contexts.md).

## Java and SQL type compatibility

Use types supported by your specific SQLJ translator/JDBC runtime. Common examples
include `int` for INTEGER, `long` for BIGINT, `short` for SMALLINT,
`java.math.BigDecimal` for DECIMAL and String for character data. Temporal, LOB,
nullable and conversion behavior requires the vendor/runtime contract. JDBC
support for `java.time` does not establish SQLJ translator support for those types.
The plugin does not publish a universal automatic DB2-to-Java mapping table.

Java checks require the [enhanced view and project setup](ide-features.md).
Database syntax/schema assistance is supplied by optional IDE SQL tooling, not a
complete compiler embedded in this plugin.
