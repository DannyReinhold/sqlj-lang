<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Walk through the generated DB2 application

The runnable starting point is the installed plugin's **New Project wizard**.
Follow [getting started](../user-guide/getting-started.md) to install the package,
create/import the demo and complete its first DB2 startup. No private source clone
or separate example download is required.

## Follow one request through the code

1. Open `ProductController.java` under `src/main/java` and find its repository use.
   Ctrl+B on `ProductRepository` should open `ProductRepository.sqlj` under
   `src/main/sqlj`, rather than requiring you to edit generated Java.
2. In the SQLJ repository, inspect the named iterator declaration. Its columns
   become modeled Java accessor methods for completion and navigation.
3. Follow `findAll()`: the repository obtains a JDBC connection from the configured
   DataSource, wraps it in a SQLJ context, executes a query into an iterator,
   traverses rows and creates Product objects. The generated code shows its cleanup.
4. Find a host bind such as `:targetCategory` or `:newStock`. Ctrl+B should navigate
   to its Java declaration. In a disposable copy, try an invalid member name and
   confirm highlighting, then undo the change.
5. Run the application and open [http://localhost:8080](http://localhost:8080).
   Browse the products and exercise a stock update. Flyway supplied the schema
   and seed records; SQLJ did not create them merely by opening the file.

## Understand the build boundary

The `.sqlj` file is the source you edit. Maven invokes the IBM translator to create
Java and SQLJ profile resources, compiles Java and installs source maps at
`process-classes`. IDE assistance can work before translation, but execution needs
the complete build/runtime. The starter POM explicitly translates its repository
input; adding another `.sqlj` requires extending that build input configuration.

For an IDE run/debug select **SQLJ Demo (Maven + SMAP)**. To package from a terminal:

```powershell
mvn clean package
```

Use the [debugger walkthrough](../user-guide/debugging-checks.md) to try stepping,
watches, conditions and logpoints in the application.

## Make one small SQLJ change

In a disposable demo copy, change a SQL query's ordering while keeping the selected
columns compatible with the iterator, then rebuild and verify the page result.
Do not change generated Java. If a build rejects the SQL, retain its first
translator/database message; the IDE is not the final authority on DB2 semantics.

Continue with [iterators](working-with-iterators.md) and
[connection/execution contexts](db2-connection-and-contexts.md).
