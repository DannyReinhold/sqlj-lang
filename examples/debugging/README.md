<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# SQLJ stepping, stack frames and local variables

Use a current wizard project and the **SQLJ Demo (Maven + SMAP)** configuration.
This is an installed-plugin acceptance procedure, not an editor-only fixture.
Use the development DB2 container. Do not transfer it into a production deployment.
The application need not otherwise depend on Spring Boot: for other launch types,
follow [the Java/Spring/Liberty setup](../../docs/user-guide/debugging-project-types.md).

Record the plugin/IDE/JDK versions and whether each observation passed. Breakpoint
hits and automatic builds have been confirmed in the wizard; the following
stepping/evaluation results still need separate installed-plugin acceptance.

## 1. Step Over and initialization

Set a SQLJ breakpoint on the `List<Product> result = new ArrayList<Product>();`
statement in `ProductRepository.findAll()`. Debug the application and open the
product page so the method is executed. Select the repository frame under
**Threads & Variables** in the Debug window (`Alt+5`).

Before the declaration executes, `this` should be available but `result` can be
absent: a breakpoint suspends before executing the selected statement. Press
**Step Over** (`F8`). `result` should then appear as an empty list in that frame.
Repeat for `conn` and `ctx`: inspect them after their initializers have executed.
`this.dataSource` is a field, not a local variable, and is inspected under `this`.

Set a breakpoint on the opening `#sql [ctx] iter = {` line. Resume to it. Press F8
and check whether the next stop is after that source statement, rather than a
series of stops on generated implementation lines. The query may take time while
it runs; Step Over does not bypass database execution. After it completes, the
iterator should be available. Record any generated-Java stop or repeated source
line rather than treating the scenario as passed.

The plugin's runtime filter now applies only to `sqlj.runtime.*`,
`com.ibm.db2.jcc.sqlj.*` and `com.ibm.db2.jcc.am.*`. It does not globally force Step
Into or hide ordinary user, Spring or Liberty methods. IntelliJ's own stepping
settings may separately skip JDK methods, constructors or synthetic methods.

## 2. A controlled user-method call inside the SQLJ file

In a disposable copy of the example, add this helper **inside** the existing
`ProductRepository` class:

```java
private static int debugRowCount(int count) {
    int offset = 1;
    int visibleCount = count + offset;
    return visibleCount;
}
```

Immediately before `return result;` in `findAll()`, add:

```java
int inspectedCount = debugRowCount(result.size());
```

The existing SQLJ input remains `ProductRepository.sqlj`, so the wizard's translator
execution includes the change. Do not merely add a second `.sqlj` input and assume
the wizard's baseline translation command automatically discovers it.

Stop the old debug session, set a breakpoint on the new call and start Debug again.
The configured Maven preparation must rebuild and install fresh metadata.

| Action / stop | Expected observation |
| --- | --- |
| Break on `inspectedCount = ...` | `result` contains the fetched rows; `inspectedCount` may not yet exist |
| F8 at that call | Remain in `findAll()` and reach the next executable statement; `inspectedCount == result.size() + 1` |
| On a new invocation, F7 at that call | Enter the user helper; IntelliJ may first enter `result.size()` depending on its stepping filters |
| Helper entry | Parameter `count` is available; `offset` is not initialized yet |
| F8 after `int offset = 1;` | `offset == 1` in the helper frame |
| Select caller `findAll()` in the stack | Caller locals such as `result` are displayed, rather than the helper's locals |
| Select helper again | `count` and initialized helper locals are displayed |
| Step Out (`Shift+F8`) | Return to the caller; a stop at the call line before assignment is possible |
| F8 to the following statement if necessary | `inspectedCount` is now assigned and has the expected value |

Variables are scoped to the selected stack frame and execution point. A local
that is absent before its initialization or after leaving its block is not by
itself a debugger defect. Reacquire values after each step; old frame objects are
invalid after resuming execution.

## 3. Runtime and unrelated Java stepping

When stepping into a SQLJ operation, the extra filter should step out of the
listed SQLJ/DB2 runtime classes. It must not hide your own helper methods or change
an ordinary Java application's F8 into F7. Check a regular `.java` call in the same
project as a control. User-defined iterator methods and generated wrappers are
not all automatically classified as runtime; record the actual stop if one appears.

Use IntelliJ's Force Step Into / ignore-filter action if deliberately inspecting
runtime internals. Record the IDE's other stepping filters when comparing results.

## 4. Watches and evaluation: separate follow-up

After the locals checks, optionally try `count`, `offset`, `count + offset` and
`this.dataSource` in their appropriate frames using Watches or Evaluate Expression.
Report whether parsing, variable lookup and actual evaluation each work.
The current patch proves JDI locals/frame access, not full IDE expression evaluation.
Do not infer evaluation support merely because the Variables pane shows a value.

## Automated evidence and remaining limits

`SqljDebugSteppingJdiTest` launches a real child JVM with a controlled SMAP installed
by the actual starter helper. It checks mapped Step Over/Into/Out, a multi-line
generated block, runtime skipping, parameters, initialized locals and the caller
frame. The runtime class is a tiny test fixture, not the IBM runtime. Policy tests
cover exact package boundaries and preserve user/Spring/Liberty frames.

These checks do not exercise IntelliJ's complete UI, real DB2 translation/execution,
Liberty, runtime expression evaluation or every generated iterator wrapper.
That is why this manual scenario remains a separate acceptance gate.
