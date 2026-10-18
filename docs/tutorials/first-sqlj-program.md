<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Tutorial: Your First SQLJ Program

In this tutorial, you will create and run a complete SQLJ application that creates a table, inserts records with host variables, and reads them back.

---

## Step 1: Create the Project File

Create a file named `HelloSqlj.sqlj` under `src/main/sqlj/org/example/HelloSqlj.sqlj`:

```java
package org.example;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import sqlj.runtime.ref.DefaultContext;

public class HelloSqlj {

    // Define a named iterator for querying greeting records
    #sql public static iterator MessageIter(int id, String recipient, String message);

    public static void main(String[] args) {
        String url = "jdbc:db2://localhost:50000/SAMPLE";
        String user = "db2inst1";
        String pass = "password";

        try (Connection conn = DriverManager.getConnection(url, user, pass)) {
            // Set default SQLJ connection context
            DefaultContext.setDefaultContext(new DefaultContext(conn));

            // 1. Create table
            #sql {
                CREATE TABLE greetings (
                    id INT NOT NULL PRIMARY KEY,
                    recipient VARCHAR(100),
                    message VARCHAR(255)
                )
            };
            System.out.println("Table 'greetings' created successfully.");

            // 2. Insert rows using host variables
            insertGreeting(1, "Alice", "Welcome to SQLJ!");
            insertGreeting(2, "Bob", "Static SQL makes Java fast and reliable.");

            // 3. Query rows using single-row SELECT INTO
            int count = 0;
            #sql { SELECT COUNT(*) INTO :count FROM greetings };
            System.out.println("Total greetings in database: " + count);

            // 4. Query all rows using Named Iterator
            MessageIter iter;
            #sql iter = { SELECT id, recipient, message FROM greetings ORDER BY id };

            while (iter.next()) {
                System.out.printf("[%d] To %s: %s%n",
                    iter.id(), iter.recipient(), iter.message());
            }
            iter.close();

            // 5. Clean up
            #sql { DROP TABLE greetings };
            System.out.println("Table dropped. Tutorial complete!");

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private static void insertGreeting(int id, String recipient, String msg) throws SQLException {
        #sql {
            INSERT INTO greetings (id, recipient, message)
            VALUES (:id, :recipient, :msg)
        };
    }
}
```

---

## Step 2: What Happened Here?

1. **`DefaultContext.setDefaultContext(...)`**: Associates the active JDBC connection with all `#sql` statements that do not specify an explicit context qualifier.
2. **`#sql { CREATE TABLE ... }`**: DDL executed directly without `Statement.executeUpdate()` string formatting.
3. **`:id, :recipient, :msg`**: Host variables automatically bound by name without positional `?` placeholder indexing.
4. **`SELECT COUNT(*) INTO :count`**: Direct assignment into local Java integer variable.
5. **`MessageIter`**: Strongly typed cursor with generated `id()`, `recipient()`, and `message()` accessors.

---

## Next Steps

- Learn about advanced iterators in [Working with SQLJ Iterators](working-with-iterators.md).
- Learn about multi-connection handling in [DB2 Connections & Execution Contexts](db2-connection-and-contexts.md).
