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

Contexts can specify connection properties and isolation parameters at declaration time:

```java
#sql public context ReadOnlyReportingContext with (
    readOnly = true,
    isolation = "TRANSACTION_READ_UNCOMMITTED"
);
```

---

## 4. Best Practices

1. **Always close contexts**: Closing a context releases underlying prepared statement caches.
2. **Handle transaction boundaries explicitly**: Use `conn.commit()` / `conn.rollback()` or configure autocommit behavior on the connection context.
3. **Use DefaultContext for simple services**: For single-datasource applications, setting `DefaultContext.setDefaultContext(...)` avoids repeating qualifiers on every statement.
