<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# SQLJ Tutorials

Learn SQLJ step-by-step through practical, runnable examples.

---

## Tutorial Tracks

### 1. [Your First SQLJ Program](first-sqlj-program.md)
Create a standalone SQLJ program, establish a database connection, create tables, insert records using host variables, and query single-row results.

### 2. [Working with SQLJ Iterators](working-with-iterators.md)
Master both **Named Iterators** and **Positional Iterators** for multi-row queries, cursor lifecycle management, null value handling, and batch operations.

### 3. [DB2 Connections & Execution Contexts](db2-connection-and-contexts.md)
Manage multiple database connections simultaneously, declare explicit execution contexts, handle distributed transactions, and optimize connection pooling.

---

## Quick Reference Summary

| Concept | Syntax Example | Best Used For |
|---|---|---|
| **DML Statement** | `#sql { INSERT INTO t VALUES (:val) };` | Inserts, updates, deletions |
| **Named Iterator** | `#sql iterator MyIter(int id, String name);` | Multi-column queries with typed accessors |
| **Positional Iterator** | `#sql iterator PosIter(int, String);` | Lightweight sequential cursor fetches |
| **Context Routing** | `#sql [auditCtx] { INSERT INTO audit_log ... };` | Multi-database / isolated transactions |
