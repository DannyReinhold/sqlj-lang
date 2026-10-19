<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Connection and execution contexts

These fragments assume a configured DB2 SQLJ application, runtime dependencies
and an existing JDBC connection. Start with the [wizard walkthrough](first-sqlj-program.md)
for a runnable baseline. Application transaction management remains outside the plugin.

## Wrap an application-owned JDBC connection

A connection context associates SQLJ statements with a JDBC connection. For
example, inside a method that receives a valid `java.sql.Connection connection`
and handles SQLException:

```java
#sql public context AppContext;
```

Declare the context at a legal top-level/nested declaration location; then use it
inside the method:

```java
AppContext context = new AppContext(connection);
try {
    int count = 0;
    #sql [context] { SELECT COUNT(*) INTO :count FROM employee };
    System.out.println(count);
} finally {
    context.close(sqlj.runtime.ConnectionContext.KEEP_CONNECTION);
}
```

`KEEP_CONNECTION` closes context resources while leaving the supplied JDBC
connection under its owner's control. Follow your actual runtime/pool's ownership
contract; do not casually close framework-managed connections. Do not set a
global DefaultContext per web request as a substitute for request/transaction
scoping. The generated demo provides its own explicit-context example.

## Supply execution state separately

A `sqlj.runtime.ExecutionContext` controls execution-related state. In a method
with suitable exception handling and a valid connection context:

```java
sqlj.runtime.ExecutionContext execution = context.getExecutionContext();
int count = 0;
#sql [context, execution] { SELECT COUNT(*) INTO :count FROM employee };
```

The editor resolves these Java expressions and checks connection/execution roles
when runtime types are known. It does not create transactions or verify that a
runtime connection is live. Single-entry qualifiers can refer to either role as
supported by the IBM translator. See [IBM's context-clause reference](https://www.ibm.com/docs/en/db2/11.1.0?topic=information-sqlj-context-clause).

## JNDI DataSource variant

```java
#sql public context ReportingContext with (dataSource = "jdbc/reporting");
```

This form models a different constructor family, including a parameterless
constructor for an existing JNDI resource. An application server must provision
that resource first. The plugin does not create it or configure pooling.
The wizard container demo does not require JNDI; this is an additional API/editor
scenario. Configure the server's JNDI resource separately for a Liberty deployment.

## Transactions and multiple databases

Keep transaction ownership in your application's existing layer. Calling commit
on one JDBC connection does not atomically commit a second database. The plugin
provides neither a transaction manager nor XA coordination. For Spring/Liberty,
follow the established transaction/pool configuration and release connections
through the framework's prescribed mechanism. Custom `with` attributes do not
replace those settings or automatically set JDBC isolation/read-only flags.
