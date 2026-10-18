---
title: IntelliJ plugin lessons learned
---

<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Building a language that feels native in IntelliJ

A useful DSL plugin must understand more than syntax. It must preserve the meaning
of the user's source while cooperating with the IDE's type system, indexes,
debugger and project model. Embedding Java makes that cooperation especially
important: users expect ordinary Java assistance inside a language-specific shell.

This collection records engineering lessons from Vernac's implementation. It is
intended for contributors and developers building other language plugins, including
a possible SQLJ integration. It explains mechanisms and trade-offs, not a new
language contract or a promise of support for every IntelliJ version.

## Read by problem

| Your problem | Start here |
| --- | --- |
| Decide what belongs in LSP, PSI or Java injection | [Language and Java architecture](language-and-java.md#divide-responsibilities-without-dividing-semantics) |
| Java looks correct but types, factories or completion are missing | [Resolution and completion](language-and-java.md#resolution-is-not-completion) |
| Ctrl+B works but Ctrl-hover has no link feedback | [References and navigation](language-and-java.md#navigation-needs-references-not-only-actions) |
| Broken or unsaved code disables assistance | [Editor recovery](language-and-java.md#editing-is-not-a-successful-build) |
| Breakpoints work but expressions are red or cannot complete | [Debugger contexts](debugging-and-verification.md#navigation-and-evaluation-need-different-contexts) |
| Decide what a debugger may hide or evaluate | [Honest presentation](debugging-and-verification.md#make-generated-code-less-distracting-without-hiding-behavior) |
| Plan a Java/SQLJ language plugin | [Applying the lessons to SQLJ](sqlj-design-checklist.md) |
| Detailed SQLJ plugin roadmap & milestones | [SQLJ Development Plan](development-plan.md) |

## Platform vocabulary

| Term | Meaning here |
| --- | --- |
| PSI | IntelliJ's structured program model, used for references, types and code insight |
| LSP | Language Server Protocol; a portable editor/server interface |
| VFS | IntelliJ's virtual file system and its change notifications |
| EDT | Event Dispatch Thread, responsible for UI work; long tasks must not block it |
| JDI | Java Debug Interface, used to inspect/control a real JVM |
| Injection/projection | Java context assembled from mapped DSL ranges plus synthetic scaffolding |

## Evidence and current boundaries

The observations come from compiler tests, exact-SDK compilation, real-JVM/JDI
probes and user acceptance in IntelliJ IDEA Ultimate **262.10968.63**, during
**0.5.0-SNAPSHOT** development. Those forms of evidence answer different questions;
none alone establishes complete IDE correctness. See the
[verification ladder](debugging-and-verification.md#use-a-verification-ladder).

Native debugging, expression evaluation, conditions/logpoints and the reviewed
Java/DSL navigation paths have been exercised in the installed IDE. Live,
pre-generation Java APIs currently cover ordinary local value objects only. The
short-name-cache connection initially failed in the installed IDE. A context report
isolated a host/injected-file scope-union bug; its retest confirmed class-name candidates.
A subsequent package bridge targets short insertion and static-member completion, with
native PSI regressions but popup acceptance still pending. Do not interpret component
tests as verified support for all completion contexts.

Cross-language rename, Hot Swap, remote debugging and debugging dependency-module
sources remain separate workstreams. A read-only target or a source-map sidecar
does not implement them automatically.

## How to extend this collection

Record the symptom, responsible IDE layer, invariant, remedy and remaining test
boundary. Link to the implementation notes instead of duplicating their full API
inventory. Keep private customer code, credentials, transient logs and chat
transcripts out of these pages. When an assumption changes, update the lesson's
status as well as the code.

For the user experience, read the [User Guide](../user-guide/index.md), [IDE Features Guide](../user-guide/ide-features.md), and [Tutorials](../tutorials/index.md). For build workflows, local installation and codebase walkthrough, explore the [Maintainer Documentation](../maintainers/index.md). For other topics, open the [documentation map](../documentation-map.md).
