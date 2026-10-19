<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Debugger walkthrough: stepping, watches and breakpoint expressions

Use a current wizard project and the **SQLJ Demo (Maven + SMAP)** configuration.
The examples below show how to use the debugger in the generated application.
Use the development DB2 container. Do not transfer it into a production deployment.
The application need not otherwise depend on Spring Boot: for other launch types,
follow [the Java/Spring/Liberty setup](debugging-project-types.md).

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

## 4. Watches and Evaluate Expression

Use the `debugRowCount` helper from section 2. Debug again with a breakpoint before
`int visibleCount = count + offset;`, so `offset` has already been initialized.
Select the helper frame. In the Debug window add these Watches, or open
**Run > Debugging Actions > Evaluate Expression** (`Alt+F8`):

| Java expression | Expected result in the helper frame |
| --- | --- |
| `count` | The row count passed by the caller |
| `offset` | `1` |
| `count + offset` | The row count plus one |
| `visibleCount` before its initializer executes | Not yet available; check again after F8 |
| `visibleCount` after F8 | The row count plus one |

Enter **Java expressions**, without a SQLJ host colon or a `#sql` wrapper.
`:(count + offset)` is SQLJ source syntax, not the text to enter in the Java evaluator.

While still suspended, select the caller's `findAll()` frame. `result` and
`this.dataSource` should now be available; helper locals such as `offset` belong
to the helper frame. Switch back and verify the watches are rebound to that frame.
The helper is static, so `this` is unavailable there; this is expected Java behavior.

For shadowing, optionally add `private int count = 99;` to the class and make the
helper an instance method by removing `static`. Rebuild through the configured
Maven preparation, stop in the helper and compare `count` (parameter) with
`this.count` (`99`). The IDE must not silently resolve the parameter to the field.
Remove the temporary instrumentation when the check is complete.

Record Watches, the Evaluate Expression dialog, completion in that dialog and
actual execution separately. A resolved PSI expression or visible JDI local alone
does not establish that every native evaluator operation works. Conditions and
log expressions use the same preparation hooks but need their own live check.

## 5. Conditional breakpoints with deterministic hits

For this check, use a disposable copy of `ProductRepository.sqlj`. Add the following
members inside its class:

```java
private int threshold = 99;

private int debugConditional(int value) {
    int threshold = 1;
    int adjusted = value + threshold;
    return adjusted;
}
```

Before `return result;` in `findAll()`, add:

```java
for (int debugIndex = 0; debugIndex < 3; debugIndex++) {
    debugConditional(debugIndex);
}
```

The loop is executed once per invocation of `findAll()`. Stop the old session and
launch through the Maven/SMAP configuration so translation and maps include these
source edits. Temporarily disable other test breakpoints individually to avoid
confusing their stops with this scenario; keep the new breakpoint enabled.

Set a SQLJ breakpoint on `int adjusted = value + threshold;`. Right-click its
editor gutter symbol, or open **Run > View Breakpoints** (`Ctrl+Shift+F8`) and
select it. Enable **Condition** and enter this Java boolean expression:

```java
value == 1 && threshold == 1 && this.threshold == 99
```

Keep **Suspend: All** for this first check. Open the product page exactly once:

- The invocation with `value == 0` must not suspend there.
- The invocation with `value == 1` should suspend there. Inspect `value`, the local
  `threshold` (`1`) and `this.threshold` (`99`). `adjusted` is not initialized yet.
- Resume with F9. The invocation with `value == 2` must not suspend there.

The parameter/local/field distinction is intentional: a condition evaluated with
only a class-level context can confuse these names. Repeating the request repeats
the three invocations, so a subsequent matching stop is expected.

## 6. Non-suspending log expressions

On that same breakpoint, remove/disable its condition, **uncheck Suspend**, and
check **Evaluate and log**. Enter a Java expression that returns the text to print:

```java
"SQLJ probe value=" + value + ", local=" + threshold + ", field=" + this.threshold
```

Open the product page once. The application should complete the request without
an interactive stop, while the debugger Console contains three messages for
values `0`, `1`, `2`, each with `local=1` and `field=99`. Distinguish these messages
from the optional **Breakpoint hit message** and normal application logs.
Unchecking Suspend is equivalent to the breakpoint's native **Suspend: None**
policy; the debugger may still suspend internally while evaluating the expression.

Then enable condition `value == 1` again, retaining the log action and no interactive
suspension. One request should now emit only the message for `value=1`. Clear the
condition again to verify logging remains enabled independently.

For error feedback, temporarily try `missingValue == 1` as the condition. The name
is unknown in this scope; record the native debugger error/status rather than
calling the check successful because the application happens to continue. Restore
the valid expression afterward. Expression errors and non-boolean conditions are
handled by IntelliJ's Java evaluator, not by a custom SQLJ condition parser.

Conditions and log expressions can be stored with **Java** as their expression
language. They must still use the SQLJ lexical context. Existing expressions whose
language is inferred from SQLJ sources remain supported. Enter Java expressions
without host-variable colons or `#sql` wrappers in either mode.

Remove the temporary helper methods when you have finished exploring the debugger.
