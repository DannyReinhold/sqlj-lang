<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Tutorial: Working with SQLJ Iterators

These fragments compare named and positional result traversal in an already configured DB2 SQLJ project. They assume the illustrated tables, an active connection context and SQLException handling. They are not standalone runnable projects; start with the [wizard walkthrough](first-sqlj-program.md). Editor modeling does not validate every query/result type.

---

## 1. Named Iterators

A named iterator maps SQL result columns to accessor methods by matching column names or aliases.

### Example: Querying Customer Orders

```java
package org.example.sales;

import java.math.BigDecimal;
import java.sql.Date;
import java.sql.SQLException;

public class OrderRepository {

    // Declaring Named Iterator with column types
    #sql public static iterator OrderSummaryIter (
        long orderId,
        String customerName,
        Date orderDate,
        BigDecimal totalAmount
    );

    public void printLargeOrders(BigDecimal threshold) throws SQLException {
        OrderSummaryIter iter;

        #sql iter = {
            SELECT o.order_id AS orderId,
                   c.name AS customerName,
                   o.order_date AS orderDate,
                   o.total_amount AS totalAmount
            FROM orders o
            JOIN customers c ON o.customer_id = c.id
            WHERE o.total_amount >= :threshold
            ORDER BY o.total_amount DESC
        };

        try {
            while (iter.next()) {
                System.out.printf("Order #%d | %s | %s | $%.2f%n",
                    iter.orderId(),
                    iter.customerName(),
                    iter.orderDate(),
                    iter.totalAmount()
                );
            }
        } finally {
            iter.close();
        }
    }
}
```

---

## 2. Positional Iterators

Positional iterators map query results by column order (index 1, 2, 3...) rather than names. You fetch rows into local variables using `#sql { FETCH :iter INTO :var1, :var2 }`.

### Example: Positional Cursor Traversal

```java
package org.example.sales;

import java.sql.SQLException;

public class InventoryService {

    // Declaring positional iterator with types only
    #sql iterator StockIter (String, int, double);

    public void auditInventory() throws SQLException {
        StockIter stockIter;

        #sql stockIter = {
            SELECT sku, quantity, unit_price
            FROM inventory
            WHERE quantity < 10
        };

        String sku = null;
        int qty = 0;
        double price = 0.0;

        try {
            while (true) {
                #sql { FETCH :stockIter INTO :sku, :qty, :price };

                // endFetch() indicates cursor exhaustion
                if (stockIter.endFetch()) {
                    break;
                }

                System.out.printf("Low stock warning: SKU %s, Qty: %d, Price: $%.2f%n",
                    sku, qty, price);
            }
        } finally {
            stockIter.close();
        }
    }
}
```

---

## 3. Comparison: Named vs Positional Iterators

| Feature | Named Iterator | Positional Iterator |
|---|---|---|
| **Declaration** | `#sql iterator Name(Type col1, Type col2);` | `#sql iterator Name(Type, Type);` |
| **Row Navigation** | `iter.next()` | `#sql { FETCH :iter INTO :v1, :v2 };` |
| **Value Access** | Accessor method (e.g. `iter.col1()`) | Directly into bound variables |
| **Column Matching** | Name / Alias matching | Strictly column order |
| **Best For** | Business logic, readable services | Explicit positional result assignment |

---

## Next Steps

- Explore [DB2 Connections & Execution Contexts](db2-connection-and-contexts.md) for custom transaction boundaries.
