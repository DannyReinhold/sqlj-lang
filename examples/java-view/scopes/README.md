<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Host binding scope probe

Open ScopeProbe.sqlj in the test module. This is intentionally not a translatable
program: ended, later and instanceField are negative scope tests at their query
positions. Keep the experimental marker for native Java editing; the legacy
projection also uses the improved lexical lookup.

- Ctrl+B on :shadowed must navigate to the local declaration inside its block.
- :employeeId and :before must resolve to their original parameter/local name.
- :index and :input must remain complete identifiers and resolve normally.
- :ended and :later must remain unresolved; host-variable proposals must not include
  variables from an ended block, another method or a later declaration.
- In inspectStatic, argument and staticField resolve; instanceField must not.
- Repeat after an edit/save. A missing Java scope result must not be replaced by
  guessed declarations found through whole-file text scanning.

The same lexical resolver supplies host reference variants and the custom SQLJ
completion contributor. The Database Tools SQL-injection completion route still
requires separate integration checks; this patch does not claim completion in all
DB2/injected-SQL contexts. Arbitrary parenthesized Java host expressions, array/
index expressions, null-indicator relationships and OUT/INOUT write/type checking
remain separate semantic work. Recognizing the variable name `index` is not full
support for every SQLJ index/indicator construct.
