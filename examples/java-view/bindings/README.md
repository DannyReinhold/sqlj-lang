<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Explicit and implicit write targets

Copy WriteTargetProbe.sqlj to src/main/sqlj/examples/bindings in the manual test
project. Keep its shared-view marker. No DB2 connection or translation is needed.
The fixture deliberately contains errors and a placeholder procedure; it is not
a runnable database sample.

| Target | Explicit OUT / INOUT expectation |
| --- | --- |
| Local variable / nonfinal parameter | Accepted |
| Writable Java field | Accepted |
| Array element | Accepted |
| Element of a final array reference | Accepted: the array reference is not being replaced |
| Writable field through a final receiver | Accepted: the receiver is not being replaced |
| Parenthesized variable | Accepted |
| Method call, literal, arithmetic expression | ERROR: not a writable target |
| Initialized final variable / final parameter | ERROR: final target |
| Array length | ERROR: final array length |
| Unknown/inaccessible/malformed Java expression | Existing Java error; no speculative extra target error |

Inputs (`IN`) remain unrestricted by this write-target check. Compare the valid
method call/final-variable inputs with the negative output cases.

Fix errors in place: change OUT to IN for the method/literal/arithmetic probes;
remove final from fixed; replace values.length with values[index]. The associated
write-target errors must disappear without reopening. Verify completion and
navigation still work in the same file, including array indices.

## Scope and limits

Explicit OUT/INOUT is checked on simple names and parenthesized Java hosts in
marked shared-view files. Implicit output is additionally recognized in the
host target lists of top-level SELECT INTO, VALUES INTO and FETCH INTO clauses.
This does not inspect DB2 procedure signatures, check general Java/SQL type
conversions or validate the entire translator toolchain. Indicator support is
described below.

Blank final OUT variables are deliberately deferred: their legality depends on
assignment history and Java context. Final INOUT variables are rejected because
they must be read and written. SQLJ-aware definite assignment, capture rules for
lambdas, and reads/writes after SQLJ clauses remain future work. A green write-
target check is not proof that a translator accepts the complete clause.

Rules are based on contextual Java PSI, not SQL text heuristics. Syntax/Java
resolution errors are handled before write-target checks. Tests include correction,
final parameters, arrays and fields behind final references.

References:
- [IBM: Variables in SQLJ applications](https://www.ibm.com/docs/en/db2/11.1?topic=programming-variables-in-sqlj-applications)
- [SQLJ language basics: host modes and assignable expressions](https://docs.oracle.com/en/database/oracle/oracle-database/19/jsqlj/basic-language-features.html)

## Implicit INTO targets

Copy ImplicitWriteTargetProbe.sqlj alongside the explicit probe. Its first four
clauses must have no write-target errors. The following clauses deliberately
write into initialized final variables or a method-call result and must show
ERRORs without an explicit OUT marker. For FETCH, the iterator before INTO is
an input; this fixture uses Object as a placeholder because iterator-type and
DB schema checks are not part of this scenario.

Replace the invalid final targets with output and the method call with
values[0]. Their write-target errors must disappear in the same editor.

Classification walks SQLJ PSI tokens, treats each Java host atomically, ignores
comments/quoted text, and tracks SQL parentheses. Only the first top-level INTO
of a clause starting with SELECT, VALUES or FETCH is considered. FROM/WHERE and
other SQL tokens end the recognized host target list. Explicit mode markers take
precedence; contradictory explicit modes are not diagnosed in this step.

Conservative limits: CTE/WITH queries, compound SQL, SET assignments, nested SQL
statements and non-host INTO targets are not inferred.
These cases retain explicit-mode checks; absence of an error is not evidence of
complete SQL semantic validation. No Database Tools dependency is required.

[IBM positional-iterator FETCH example](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=applications-using-positioned-iterator-in-sqlj-application)

## DB2 indicator pairs

Copy IndicatorProbe.sqlj to the same manual test source package. The supported
IBM form is adjacent `:host :indicator`, optionally separated by whitespace and
comments. Each pair has one indicator; commas/operators/SQL tokens break the
association. The primary and indicator may be parenthesized Java expressions. Complex indicator
handling is an IDE prototype pending translator checks; see below. No parser/token changes are needed.

The indicator must resolve to primitive Java short. int and boxed Short receive
an ERROR; unresolved names retain their existing host-reference diagnostic.
Indicators inherit OUT/INOUT from their primary, including recognized implicit
INTO targets. Otherwise they are treated as inputs. Thus final short input
constants are accepted, but initialized final output indicators are rejected.

Acceptance:
- Navigate from indicator/secondIndicator to the local short declarations.
- The four negative probes must report type or final-write ERRORs.
- Change wrongType from int to short and boxedIndicator from Short to short:
  the type errors must disappear.
- Replace constantIndicator in output probes with indicator: write errors disappear.
- Confirm the comma-separated int outputs and targets following a complete pair
  remain correctly classified.

The fixture is intentionally invalid and uses placeholder schema/procedure/iterator
objects; it is an editor acceptance scenario, not a runnable DB2 example.

Not yet covered: the `INDICATOR` keyword form, orphan-indicator syntax,
extended-indicator value legality,
server/bind-option capability checks, and complete SQL-derived modes for CTEs or
compound SQL. Indicator values are deliberately not guessed from constants:
DBDefault/DBUnassigned behavior depends on toolchain/server settings.

[IBM Db2: indicator variables in SQLJ applications](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=programming-indicator-variables-in-sqlj-applications)

## Complex indicators and malformed adjacency

IndicatorExpressionProbe.sqlj extends the editor scenarios with short[] element
and short field targets. The final array reference is allowed because its elements
are writable. int[] elements fail the short-type check; final output fields fail
the inherited write-target check. A computed short-valued expression has the right
type but is rejected as an output target. This prototype does not reject such a
value for an input solely because it is a computation.

Three or more adjacent hosts now produce a syntax/association ERROR on the extra
hosts. Add a comma before :next in the final query and verify that the message
disappears and :next becomes an independent output target.

Use Ctrl+B on indicators, index and holder.value; existing Java expression
references must navigate to their declarations. Temporarily shorten indicators
or index and use Ctrl+Space; the relevant local names must be offered. Try removing
the indicator's final parenthesis and confirm completion remains available.

The IBM documentation consulted explicitly demonstrates simple adjacent indicator
variables and requires short. It does not establish all complex-expression or
INDICATOR-keyword variants across translators. Complex expression handling here
is IDE-side Java semantic support, not a claim of DB2 translator acceptance.
Keep these fixtures separate from translator-ready positive samples until the
customer/container translator variants have been tested. The keyword form remains
unclassified; it is neither advertised as supported nor rejected as invalid.
