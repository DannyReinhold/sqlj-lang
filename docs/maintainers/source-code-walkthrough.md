<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Source Code & Architecture Walkthrough

This document guides maintainers and contributors through the codebase of the **SQLJ Language Support Plugin (`sqlj-lang`)**. It explains the role of each package, class, extension point, and architectural invariant.

---

## 1. Architectural Overview

The plugin bridges SQLJ language constructs (`.sqlj`) with the standard IntelliJ Java ecosystem, the Database Tooling (DataGrip/SQL dialects), and the Java Debugger (JDI).

```
+-----------------------------------------------------------------------------------+
|                                IntelliJ IDEA Platform                             |
+-----------------------------------------------------------------------------------+
         |                                |                               |
+------------------+             +------------------+             +-----------------+
|   Lexer & Parser |             | Language Inject  |             | Synthetic PSI   |
|   (Tokenization, |             | (DB2 SQL Dialect |             | (Synthetic Java |
|    AST & PSI)    |             |  in #sql blocks) |             |  Classes/Types) |
+------------------+             +------------------+             +-----------------+
         |                                |                               |
+-----------------------------------------------------------------------------------+
|                             Host Variable Resolution                              |
|               (SqljHostVariableReference <-> SqljJavaScopeResolver)               |
+-----------------------------------------------------------------------------------+
         |                                |                               |
+------------------+             +------------------+             +-----------------+
|  Code Insight &  |             | Debugger Bridge  |             | Toolchain &     |
|  Highlighting    |             | (PositionManager,|             | Project Wizard  |
|  (Annotator,     |             |  Breakpoints,    |             | (Compiler,      |
|   Completions)   |             |  Evaluation)     |             |  ModuleBuilder) |
+------------------+             +------------------+             +-----------------+
```

---

## 2. Package Tour & Responsibilities

The codebase is structured under `src/main/java/org/sqlj/lang/`:

```
org.sqlj.lang
├── SqljFileType.java                  # File type definition for *.sqlj
├── SqljLanguage.java                  # Language definition singleton
├── SqljIcons.java                     # Plugin & file type icons
├── completion/                        # Code completion contributors
├── debugger/                          # Source-level JDI debugging bridge
├── highlighting/                      # Highlighting, annotators & color settings
├── injection/                         # Multi-host SQL language injection
├── lexer/                             # Tokenization & lexer implementation
├── navigation/                        # Find usages & Goto class contributors
├── parser/                            # AST element types, parser definition & parser
├── psi/                               # PSI interface definitions
│   ├── impl/                          # AST-to-PSI element implementations
│   ├── references/                    # Host variable reference resolution & scoping
│   └── synthetic/                     # Synthetic Java classes for iterators/contexts
├── toolchain/                         # Compiler runner, diagnostics & settings
└── wizard/                            # New Project Wizard & Spring Boot template
```

---

### 2.1 Core Language & File Types (`org.sqlj.lang`)

- **`SqljLanguage.java`**: The root singleton representing the SQLJ language (`"SQLJ"`). Inherits from `com.intellij.lang.Language`.
- **`SqljFileType.java`**: Defines the `.sqlj` file association with MIME types and icon mappings.
- **`SqljIcons.java`**: Icon registry providing SVG icons for SQLJ files (`/icons/sqlj-file.svg`) and plugin branding.

---

### 2.2 Lexer & Parsing Subsystem (`lexer`, `parser`, `psi`)

SQLJ blends Java syntax with embedded `#sql` clauses. The parsing subsystem breaks sources into structured PSI trees:

- **`lexer/SqljTokenType.java` & `SqljTokenTypes.java`**: Define IElementType tokens representing `#sql` directives (`#sql`, `iterator`, `context`, `:OUT`, `:INOUT`, etc.).
- **`lexer/SqljLexer.java`**: The lexical analyzer that scans `.sqlj` files, extracting SQLJ-specific tokens while delegating surrounding Java text appropriately.
- **`parser/SqljParser.java` & `SqljParserDefinition.java`**:
  - Implements recursive descent parsing for SQLJ statements and declarations.
  - Builds AST nodes for `#sql { ... }` blocks, iterator definitions (`#sql public static iterator MyIter(...)`), context definitions (`#sql context MyCtx`), and host variable expressions (`:var`, `:(expr)`).
- **`psi/SqljFile.java`**: Root PSI file element representing a `.sqlj` file.
- **`psi/SqljStatement.java` & `psi/SqljCodeBlock.java`**: Represent embedded SQL executions and block enclosures.
- **`psi/SqljIteratorDeclaration.java` & `psi/SqljContextDeclaration.java`**: Represent iterator and context declarations.
- **`psi/SqljHostExpression.java`**: Represents bind variables (`:myParam`, `:OUT result`).

---

### 2.3 Multi-Host SQL Language Injection (`injection`)

- **`SqljLanguageInjector.java`**:
  - Implements IntelliJ's `MultiHostInjector`.
  - Injects the **DB2 SQL dialect** (or generic SQL) directly into the text ranges within `#sql { ... }` and `#sql [conn] var = { ... }`.
  - Leaves host variable ranges (`:var`) un-injected or mapped as parameter placeholders so that IntelliJ's SQL engine performs table, column, and clause completion without breaking on Java variables.

---

### 2.4 Host Variable Binding & Java Scoping (`psi.references`)

Developers use Java variables inside SQL queries via host binds (`:myVar`). The plugin provides bidirectional navigation (`Ctrl+B`) and refactoring:

- **`SqljReferenceContributor.java`**: Registers reference providers on `SqljHostExpression` elements.
- **`SqljHostVariableReference.java`**: Resolves a host variable name (e.g., `:productId`) to its corresponding Java declaration.
- **`SqljJavaScopeResolver.java`**:
  - Walks the enclosing Java/PSI scope (method parameters, local variables, class fields).
  - Handles parenthesized host expressions (`:(service.calculateId())`).
  - Supports navigation and Find Usages across Java and SQLJ files.
- **`SqljJavaVariableTarget.java`**: Target wrapper representing resolved Java fields, parameters, or local variables.

---

### 2.5 Synthetic Java PSI Classes (`psi.synthetic`)

SQLJ generates Java classes during offline translation (e.g., `ProductIterator_SJProfileKeys.java` or inner iterator classes). To provide instant code completion in Java files **before** running the SQLJ compiler:

- **`SqljElementFinder.java`**: Implements `PsiElementFinder`. Intercepts class resolution queries from IntelliJ's Java type system for synthetic iterators, synthetic contexts, and regular Java classes declared inside `.sqlj` files.
- **`SqljSyntheticIteratorClass.java`**: Creates an on-the-fly, in-memory `PsiClass` for `#sql iterator ...` declarations, providing synthetic accessor methods (`id()`, `name()`, `next()`, `close()`).
- **`SqljSyntheticContextClass.java`**: Creates synthetic context classes for `#sql context ...` declarations implementing `sqlj.runtime.ref.ConnectionContext`.
- **`SqljSyntheticJavaClass.java`**: Exposes regular Java classes (e.g. `ProductRepository`) declared inside `.sqlj` files to the IntelliJ Java PSI facade.

---

### 2.6 Code Insight, Highlighting & Navigation (`highlighting`, `completion`, `navigation`)

- **`SqljSyntaxHighlighter.java`**: Assigns text attributes for SQLJ keywords, directives, host variable prefixes, and comments.
- **`SqljAnnotator.java`**: Validates SQLJ constructs in real-time (e.g., highlighting unmatched host variables or illegal iterator modifiers).
- **`SqljColorSettingsPage.java`**: Provides customization of SQLJ color schemes under IDE Settings -> Editor -> Color Scheme -> SQLJ.
- **`SqljCompletionContributor.java`**: Context-aware completion suggestions for Java keywords, visible Java variables/parameters/fields, iterator member methods (`.next()`, column accessors), SQLJ keywords (`#sql`, `iterator`, `context`), and SQL host variables.
- **`SqljGotoDeclarationHandler.java`**: Resolves `Go to Declaration` (`Ctrl+B`) across both SQLJ host variables (`:var`), iterator/context types, and surrounding Java expressions and declarations.
- **`SqljFindUsagesProvider.java` & `SqljGotoClassContributor.java`**: Enables `Find Usages` and `Navigate to Class / Symbol` for SQLJ iterators, contexts, and Java classes.

---

### 2.7 Source-Level Debugging Bridge (`debugger`)

SQLJ source code is translated into generated Java classes before execution. The debugger bridge allows setting breakpoints directly in `.sqlj` files:

- **`SqljPositionManagerFactory.java` & `SqljPositionManager.java`**:
  - Maps source positions in `.sqlj` files to runtime class names and line numbers in translated classes (`.java`).
  - Converts JDI `Location` objects from the running JVM back to `.sqlj` source lines when pausing at breakpoints.
- **`SqljLineBreakpointType.java`**: Enables breakpoint placement on executable `.sqlj` lines.
- **`SqljCodeFragmentFactory.java`**: Enables expression evaluation (`Alt+F8`) and watch expressions in `.sqlj` breakpoint contexts using the surrounding Java scope.
- **`SqljSteppingFilter.java`**: Automatically filters out internal SQLJ runtime generated profiles and driver internals during "Step Into".

---

### 2.8 Toolchain & Compiler Integration (`toolchain`)

- **`SqljProjectSettings.java`**: Persistent project-level state (configured SQLJ translator path, DB2 environment, generated sources directory).
- **`SqljSettingsConfigurable.java`**: UI configuration page under `Settings` -> `Languages & Frameworks` -> `SQLJ & DB2 Toolchain`.
- **`SqljCompilerRunner.java`**: Invokes external SQLJ translators (`db2sqljcompiler` / `sqlj`) on project build.
- **`SqljCompilerErrorParser.java` & `SqljCompilerMessage.java`**: Parses translator stdout/stderr and maps compilation errors directly to `.sqlj` source lines.
- **`SqljGeneratedSourcesManager.java`**: Automatically registers generated Java source directories as project source roots.

---

### 2.9 Project Wizard & Template Generator (`wizard`)

- **`SqljModuleBuilder.java`**: Extends IntelliJ's `ModuleBuilder` to provide the **SQLJ Spring Boot Application** option in the New Project dialog.
- **`SqljModuleWizardStep.java`**: Configures Maven coordinates (`groupId`, `artifactId`, `version`, `packageName`) with live validation.
- **`SqljProjectTemplateGenerator.java`**: Generates a complete project skeleton:
  - Spring Boot 3.4 + Thymeleaf + Spring JDBC `pom.xml` with Flyway (`flyway-core`, `flyway-database-db2`), dual-mode SQLJ profiles, and `spring-boot-docker-compose`
  - IBM DB2 Docker configuration (`compose.yaml` with port 50000 and `SAMPLE` database)
  - Flyway migration scripts (`V1__init_schema.sql` and `V2__sample_data.sql` in `db/migration`)
  - Working Java 1.5-compatible `ProductRepository.sqlj` using typed iterators, `DefaultContext`, and host variables
  - Spring MVC Controller, record models, and Bootstrap Thymeleaf UI

---

## 3. Configuration & Extension Registration

- **`src/main/resources/META-INF/plugin.xml`**: Declares all IntelliJ extension points (`fileType`, `lang.parserDefinition`, `lang.syntaxHighlighterFactory`, `annotator`, `completion.contributor`, `languageInjector`, `psi.referenceContributor`, `elementFinder`, `debugger.positionManagerFactory`, `projectConfigurable`, `moduleBuilder`, etc.).
- **`src/main/resources/messages/SqljBundle.properties`**: Internationalized messages and UI labels.

---

## 4. Testing Framework & Test Suite Architecture

All tests reside under `src/test/java/org/sqlj/lang/` and use IntelliJ's `BasePlatformTestCase`:

| Test Class | Focus Area |
|---|---|
| `PluginConfigurationTest` | Plugin descriptor (`plugin.xml`) and bundle verification |
| `SqljLexerTest` | Tokenization of SQLJ clauses, host variables, and edge cases |
| `SqljPsiTest` | AST and PSI tree construction |
| `SqljHostVariableReferenceTest` | Host variable to Java declaration resolution |
| `SqljSyntheticClassTest` | In-memory synthetic iterator and context classes |
| `SqljLanguageInjectorTest` | DB2 SQL dialect injection into `#sql` blocks |
| `SqljCompletionTest` | Keyword and host variable code completion |
| `SqljAnnotatorTest` | Real-time error and warning annotations |
| `SqljDebuggerTest` | Position manager line mapping and breakpoint types |
| `SqljCompilerErrorParserTest` | Compiler error regex matching and line mapping |
| `SqljToolchainSettingsTest` | Settings persistence and state serialization |
| `SqljModuleBuilderTest` | New Project Wizard generation and template validation |

---

## Next Steps
- For build and execution commands, see [Building, Running & Local Installation](building-and-running.md).
- For publishing and release management, see [Releasing & Deployment](releasing-and-deployment.md).
