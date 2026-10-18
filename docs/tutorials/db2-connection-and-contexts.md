<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Tutorial: DB2 Connections & Execution Contexts

In enterprise systems, applications frequently coordinate operations across multiple data sources or require explicit transaction boundaries. This tutorial explains how to manage connections with **Connection Contexts**.

---

## 1. Declaring Context Classes

In SQLJ, connection context types are declared with `#sql context`:

```java
package org.example.banking;

// Context for primary transactions
#sql public context CoreBankingContext;

// Context for audit logs (often pointing to a separate audit database)
#sql public context AuditContext;
```

When preprocessed, SQLJ generates Java classes (`CoreBankingContext`, `AuditContext`) subclassing `sqlj.runtime.ref.ConnectionContextImpl`.

---

## 2. Using Context Qualifiers

You can direct `#sql` clauses to execute on a specific context by prefixing the statement with `[contextInstance]`.

### Example: Multi-Database Transfer and Audit

```java
package org.example.banking;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.math.BigDecimal;

public class TransferService {

    public void transferAndAudit(
        Connection coreConn,
        Connection auditConn,
        long fromAccount,
        long toAccount,
        BigDecimal amount
    ) throws SQLException {

        CoreBankingContext coreCtx = new CoreBankingContext(coreConn);
        AuditContext auditCtx = new AuditContext(auditConn);

        try {
            // 1. Deduct funds on Core Banking DB
            #sql [coreCtx] {
                UPDATE accounts
                SET balance = balance - :amount
                WHERE account_id = :fromAccount
            };

            // 2. Deposit funds on Core Banking DB
            #sql [coreCtx] {
                UPDATE accounts
                SET balance = balance + :amount
                WHERE account_id = :toAccount
            };

            // 3. Commit banking transaction
            coreConn.commit();

            // 4. Record audit entry on Audit DB
            #sql [auditCtx] {
                INSERT INTO transfer_audit_log (
                    from_acct, to_acct, transfer_amt, event_time
                ) VALUES (
                    :fromAccount, :toAccount, :amount, CURRENT TIMESTAMP
                )
            };

            auditConn.commit();

        } catch (SQLException e) {
            coreConn.rollback();
            throw e;
        } finally {
            coreCtx.close();
            auditCtx.close();
        }
    }
}
```

---

## 3. Context Configuration with `with` Clause

A DB2 context can name a separately configured JNDI DataSource:

```java
#sql public context ReportingContext with (dataSource = "jdbc/reporting");

// The JNDI resource must already exist in the application environment.
ReportingContext context = new ReportingContext();
```

This form supplies parameterless and user/password constructors in addition to
wrappers for an existing JDBC Connection or SQLJ ConnectionContext. Contexts
without dataSource use the URL-based constructor set instead. All these
constructors declare SQLException.

Custom with attributes do not automatically configure JDBC behavior. Set read-only
or transaction isolation on the actual connection where appropriate, for example:

```java
java.sql.Connection connection = context.getConnection();
connection.setReadOnly(true);
connection.setTransactionIsolation(java.sql.Connection.TRANSACTION_READ_COMMITTED);
```

Whether those settings are supported and how they interact with pooling and
transactions depends on the driver/application. The wizard's standard DB2
container example does not require JNDI setup; keep that path as the first runnable
and debuggable baseline. The JNDI variant is an additional editor/API scenario.

---

## 4. Best Practices

1. **Always close contexts**: Closing a context releases underlying prepared statement caches.
2. **Handle transaction boundaries explicitly**: Use `conn.commit()` / `conn.rollback()` or configure autocommit behavior on the connection context.
3. **Use DefaultContext for simple services**: For single-datasource applications, setting `DefaultContext.setDefaultContext(...)` avoids repeating qualifiers on every statement.
