<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Entwicklungsplan: SQLJ IntelliJ Plugin (`sqlj-lang`)

> Statushinweis (2026-10-17): Die bisherigen Häkchen dokumentieren vorhandene
> Implementierungsansätze, keine vollständig verifizierten IDE-Funktionen.
> Maßgeblich für die nächsten Schritte ist der [Integrationsfahrplan](integration-roadmap.md).

Dieses Dokument definiert den Phasen- und Entwicklungsplan für das IntelliJ IDEA Plugin **sqlj-lang** zur nahtlosen Unterstützung von SQLJ mit eingebettetem Java und DB2 SQL-Dialektunterstützung.

---

## 1. Übersicht & Rahmenbedingungen

- **Plugin-Name**: `sqlj-lang` (SQLJ Language Support for IntelliJ Platform)
- **Lizenz**: PolyForm Noncommercial 1.0.0 (kommerzielle Lizenz als Alternative)
- **Plattform-Zielversion**: IntelliJ IDEA Platform ab Build `#261.24374-151` (2026.1+ bzw. aktuelle Plattformlinie)
- **SDK & Runtime**: Java 21 LTS, Gradle mit `org.jetbrains.intellij.platform` Plugin (empfohlener Standard für JetBrains Marketplace Builds) bzw. Maven Plugin Build-Toolchain mit IntelliJ Platform Integration
- **Distribution & Repositories**:
  - Gitea (Quellrepository und CI/CD); GitHub ausschließlich für Dokumentation
  - JetBrains Marketplace (Plugin Repository mit automatisiertem Signing, Verification & Publishing)
- **Hauptanwendungsfall**: Entwicklung und Debuggen von SQLJ-Dateien (`.sqlj`) in Java-Enterprise- und Legacy-Projekten mit DB2-Datenbankunterstützung.

---

## 2. Kernanforderungen & Architekturkonzept

### 2.1 Language & File Integration
- **Dateityp**: Registrierung des Dateityps `*.sqlj` (`SqljFileType`, `SqljLanguage`).
- **Parsing- & Grammatikstrategie**:
  - SQLJ ist eine Obermenge von Java mit speziellen SQLJ-Klauseln (z.B. `#sql { ... }`, `#sql iterator ...`, `#sql context ...`).
  - Hybrides Parsing / Multi-Host Language Injection:
    - Äußere Struktur: Java PSI als Host oder SQLJ-Host-PSI mit projektivem Java-Context (`MultiHostInjector` / `LanguageInjectionSupport`).
    - Eingebettete SQL-Blöcke: Injection des IntelliJ SQL-Dialekts (`DB2LanguageDialect` aus dem Database Tools / SQL Plugin) innerhalb der `#sql { ... }`-Rümpfe.
    - Host-Bind-Variablen (`:variable`, `:(expression)`, `:OUT variable`): Exakte bidirektionale Referenzauflösung zwischen SQL-Fragmenten und umgebendem Java-Scope (Felder, Parameter, lokale Variablen).

### 2.2 Editor Features & Code Insight
- **Syntax Highlighting**:
  - Vollständiges Java-Highlighting für Java-Bereiche.
  - SQL/DB2 Highlighting für SQL-Fragmente.
  - Spezifisches Highlighting für SQLJ-Klauseln (`#sql`, `iterator`, `context`, etc.) und Bind-Variablen.
- **SQL-Syntaxprüfung (DB2)**:
  - Validierung eingebetteter SQL-Statements gegen den DB2 SQL-Grammatikbaum.
  - Bereitstellung von Error-Highlighting und Annotationen ohne den umgebenden Java-Code zu invalidieren (Recovery-Resilienz).
- **Code Completion**:
  - Java Completion innerhalb des Java-Codes und in Host-Ausdrücken (`:(myJavaMethod())`).
  - SQL Completion (Tabellen, Spalten, DB2 Keywords, Schemata) innerhalb von `#sql { ... }` bei verknüpfter Database DataSource / Dialect Context.
  - SQLJ-spezifische Klauseln und Bind-Variablen-Vervollständigung basierend auf dem lokalen Java-Scope.
- **Navigation (Go To Declaration / Usages)**:
  - Java ↔ SQLJ Referenzen: Navigation von Bind-Variablen im SQL-Fragment zur Java-Deklaration und umgekehrt.
  - Navigation zu generierten Klassen/Iteratoren vermeiden: Der Entwickler navigiert direkt zu den SQLJ-Originaldateien und -Methoden.
  - Deklarationen von SQLJ-Iteratoren (`#sql public iterator MyIter(...)`) als typsichere Java-Klassen für den restlichen Java-Code verfügbar machen (`PsiElementFinder` / synthetic Java classes).

### 2.3 Debugging (Source-Level Debugging)
- **Ziel**: Nahtloses Debuggen in `.sqlj`-Dateien ohne in generierte `.java`-Dateien abspringen zu müssen.
- **Position Mapping**:
  - Implementierung eines `PositionManager` (`SqljPositionManager` via `JavaDebugPositionManagerHelper` / `PositionManagerFactory`).
  - Abbildung von JVM-Klassen- und Bytecode-Zeilen auf die korrekten Quelltextzeilen der `.sqlj`-Datei anhand der vom SQLJ-Translator generierten Line-Number Tables oder SourceMaps / `#line`-Direktiven.
- **Breakpoints**:
  - `SqljLineBreakpointType` basierend auf `JavaLineBreakpointType`.
  - Korrekte Platzierung auf ausführbaren SQLJ- und Java-Zeilen.
- **Expression Evaluation & Context**:
  - `JavaCodeFragmentFactory` / `EvaluationContextProvider` zur Evaluierung von Ausdrücken, Watches und Breakpoint-Bedingungen im echten Java-Scope des Stackframes.
  - Vermeidung von Rekursionen in Breakpoint-Callbacks und Scope-Lookups (gemäß Lessons Learned).

---

## 3. Phasenplan

```
Phase 0: Projekt-Setup, Build-Infrastruktur & Plattform-Vorgaben
   ├── Gradle/Maven IntelliJ Platform Setup (Target Build #261.24374-151)
   ├── PolyForm-Lizenzierung, Gitea-Quellrepository & CI Workflows
   └── Plugin-Deskriptor (plugin.xml) & Marketplace Compliance

Phase 1: Lexer, Parser & Dateityp-Registrierung (Core PSI & Dialect)
   ├── FileType, Language & Icon Setup
   ├── SQLJ Grammatik (Grammar-Kit / JFlex oder ANTLR/Pratt Parser)
   └── Basis-Highlighting & Error-Toleranz (Recovery)

Phase 2: Multi-Host Language Injection & Java-Integration
   ├── Injection von DB2 SQL in #sql-Fragmente
   ├── Bidirektionales Bind-Variable-Mapping (:var <-> Java Scope)
   └── Synthetische PSI-Bereitstellung für Iteratoren/Contexts (PsiElementFinder)

Phase 3: Code Insight (Completion, Navigation & Inspektionen)
   ├── DB2 SQL-Validierung & Fehler-Recovery im Editor
   ├── Kontextsensitive Completion (Java, DB2, Bind-Variablen)
   └── Go To Definition & Find Usages (Java <-> SQLJ)

Phase 4: Debugger-Integration (PositionManager & Breakpoints)
   ├── SqljPositionManager & Source-to-Bytecode Line Mapping
   ├── SqljLineBreakpointType & Java Debugger Bridge
   └── Expression Evaluation, Watches & Bedingte Breakpoints

Phase 5: DB2 Toolchain, Compiler/Translator Bridge & Build-System
   ├── Integration mit SQLJ-Translators (IBM DB2 sqlj / db2sqlj)
   ├── Mapping von Compiler-Meldungen auf Quellzeilen
   └── Generated Sources Root Registration & Ownership Model

Phase 6: Verifikation, Marketplace Readiness & Release
   ├── Test-Pyramide (Unit-, PSI-, Injection-, JDI-Debugger- & UI-Tests)
   ├── Plugin Verifier (#261.24374-151 Kompatibilitätsprüfung)
   ├── Dokumentation, Screenshots, Marketplace-Assets & CI Deployment
   └── Release v1.0.0 auf JetBrains Marketplace & GitHub
```

---

## 4. Detaillierte Meilensteine

### Phase 0: Projekt-Setup & Plattform-Konfiguration
- [x] **Build-System**: Migration bzw. Konfiguration des Builds mit `intellij-platform-gradle-plugin` (2.x) für IntelliJ IDEA Platform `#261.24374-151` / Java 21 LTS inkl. Gradle Wrapper.
- [x] **SDK & Kompatibilität**:
  - Target-Plattform: Build `#261.24374-151` (`since-build="261.24374"`).
  - Java 21 Target Bytecode & Toolchains.
- [x] **Lizenz & Legal**:
  - `LICENSE`-Datei (PolyForm Noncommercial 1.0.0; Doku CC BY 4.0).
  - `NOTICE` und Lizenzinformationen.
- [x] **GitHub & CI/CD**:
  - GitHub Actions Workflow für Build, Test, Plugin Verification (`.github/workflows/build.yml`).
  - Marketplace Publication Workflow mit Secrets (`.gitea/workflows/release.yml`).
  - `.gitignore` für Gradle & IntelliJ.
- [x] **Marketplace Metadaten & Plugin-Deskriptor**:
  - `plugin.xml` mit ausführlicher HTML-Beschreibung, Change Notes, Vendor Info, Documentation URL, Issue Tracker URL.
  - Abhängigkeiten: `com.intellij.modules.platform`, `com.intellij.java`, optional `com.intellij.database` (`sqlj-database.xml`).
  - Ressourcen-Bündel (`SqljBundle.properties`) und Plugin-Icon (`pluginIcon.svg`).

### Phase 1: Lexer, Parser & PSI Grundlagen
- [x] **Dateityp & Language**: `SqljFileType`, `SqljLanguage`, Icons für `.sqlj`.
- [x] **Lexer & Tokens**:
  - Lexer für SQLJ Keywords (`#sql`, `iterator`, `context`, etc.), Java-Bereiche, Bind-Variablen (`:var`, `:IN`, `:OUT`, `:INOUT`, `:(expr)`), Delimiter und SQL-Klauseln.
- [x] **Parser & PSI Struktur**:
  - AST-Knoten für SQLJ-Direktiven, Declarations (`#sql iterator ...`, `#sql context ...`), Execution Statements (`#sql [ctx] { ... }`, `#sql iter = { ... }`).
  - Vollständige PSI-Hierarchie (`SqljFile`, `SqljStatement`, `SqljIteratorDeclaration`, `SqljIteratorColumn`, `SqljContextDeclaration`, `SqljCodeBlock`, `SqljHostExpression`).
  - Fehlertolerantes Parsing: Unvollständige oder syntaktisch fehlerhafte SQL-Abschnitte zerstören die umgebende Struktur nicht.
  - Syntax Highlighting & Color Settings (`SqljSyntaxHighlighter`, `SqljColorSettingsPage`).

### Phase 2: Multi-Host Language Injection & Java-Integration
- [x] **Multi-Host Injection**:
  - Implementation von `MultiHostInjector` für `#sql { ... }` Blöcke zur Injektion des SQL Dialekts (`DB2`).
- [x] **Host-Bind Variable Resolution**:
  - Eigener Referenz-Provider (`PsiReferenceProvider`) für `:javaVar` und `:(javaExpression)` innerhalb von SQL-Fragmenten.
  - Mapping auf den umgebenden Java Lexical Scope (Methodenparameter, lokale Variablen, Klassenfelder).
- [x] **Synthetische Java-Typen (Iterators & Contexts)**:
  - `PsiElementFinder` / synthetische PSI-Klassen für deklarierte SQLJ-Iteratoren, damit normaler Java-Code Methoden wie `.next()`, `.getCol()` fehlerfrei auflösen und aufrufen kann.

### Phase 3: Code Insight (Highlighting, Completion, Navigation)
- [x] **Syntax Highlighting**:
  - `SyntaxHighlighter` für SQLJ; nahtloses Ineinandergreifen von Java Highlighting und DB2 SQL Highlighting.
- [x] **DB2 SQL Validierung**:
  - Aktivierung der SQL-Inspektionen und des Annotators (`SqljAnnotator`) innerhalb der injizierten DB2-Blöcke und Host-Variablen.
  - Anbindung an IntelliJ Database Schemas / Data Sources (sofern konfiguriert).
- [x] **Code Completion**:
  - SQL-Completion in `#sql`-Blöcken (DB2 Keywords, Tabellen, Spalten) via `SqljCompletionContributor`.
  - Java-Completion in Host-Ausdrücken und im umgebenden Code.
- [x] **Navigation**:
  - Go to Definition (Ctrl+B) von Bind-Variablen zur Java-Deklaration und zurück (Find Usages via `SqljFindUsagesProvider`).
  - Navigationsunterstützung für Iteratortypen direkt zur `#sql iterator`-Zeile in der `.sqlj`-Datei (`SqljGotoClassContributor`).

### Phase 4: Debugger-Unterstützung (Source-Level Debugging)
- [x] **PositionManager**:
  - `SqljPositionManager` registrieren (`JavaDebugPositionManagerHelper` / `PositionManagerFactory`).
  - Zuordnung von Bytecode / JVM Stackframe Locations (z.B. Zeilen im generierten Java-Code bzw. SMAP/Line-Number-Attribute) zur `.sqlj`-Originaldatei.
- [x] **Line Breakpoints**:
  - `SqljLineBreakpointType` zur einfachen Breakpoint-Setzung auf SQLJ- und Java-Zeilen im Editor.
  - Verifikation: Breakpoint-Setzen darf keine rekursiven Callbacks / StackOverflows auslösen.
- [x] **Expression Evaluation**:
  - Evaluierung von Ausdrücken (`Alt+F8`), Watches und Breakpoint-Bedingungen im Scope der aktuellen SQLJ-Quelltextposition.
  - Bereitstellung des korrekten Java-Evaluationskontexts (`JavaCodeFragmentFactory` / `SqljCodeFragmentFactory`).
- [x] **Stepping & Facade Filtering**:
  - Step Over (`F8`), Step Into (`F7`) direkt im SQLJ-Code, ohne in interne Translator-Runtime-Klassen (`sqlj.runtime.*`, `com.ibm.db2.jcc.sqlj.*`) zu steppen (`SqljSteppingFilter`).

### Phase 5: DB2 Toolchain & Build-Integration
- [x] **Compiler/Translator Runner**:
  - Toolchain-Runner & CLI-Builder für DB2 SQLJ-Translator (`sqlj` / `db2sqljcompiler`) mit flexibler Profil-Unterstützung (`SqljCompilerRunner`, `SqljProjectSettings`).
  - Unterstützung für echte installierte DB2-Instanzen, lokale DB2 in Docker / Podman und Testcontainers (`Db2EnvironmentType`).
- [x] **Error Parsing**:
  - Parser für Translator-Kompilierfehler (`SqljCompilerErrorParser`) mit Rückabbildung auf die Zeilen der `.sqlj`-Datei.
- [x] **Generated Source Roots**:
  - Automatische Registrierung und Verwaltung generierter Java-Sources (`SqljGeneratedSourcesManager`, `SqljStartupActivity`) als Generated Source Root im Projektmodell.
- [x] **Settings UI**:
  - Konfigurationsseite in den IDE-Einstellungen unter Languages -> `SQLJ & DB2 Toolchain` (`SqljSettingsConfigurable`).

### Phase 6: Verifikation, Marketplace & Veröffentlichung
- [x] **New Project Wizard & Starter Generator**:
  - `SqljModuleBuilder` und `SqljModuleWizardStep` für "New Project / Module" im IntelliJ Wizard.
  - Template-Generator (`SqljProjectTemplateGenerator`) für Spring Boot 3.4 + Thymeleaf + DB2 Docker (`compose.yaml`) und SQLJ Repository-Muster (`ProductRepository.sqlj`).
- [x] **Automatisierte Tests**:
  - Lexer/Parser Testsuite (`SqljLexerTest`, `SqljPsiTest`).
  - Language Injection & Reference Resolution Tests (`SqljLanguageInjectorTest`, `SqljHostVariableReferenceTest`, `SqljSyntheticClassTest`).
  - Code Insight Tests (Completion: `SqljCompletionTest`, Highlighting & Annotator: `SqljAnnotatorTest`, `SqljHighlighterTest`, Navigation: `SqljNavigationTest`).
  - Debugger PositionManager Tests & Breakpoint Tests (`SqljDebuggerTest`).
  - Toolchain, Settings & Error Parser Tests (`SqljCompilerErrorParserTest`, `SqljCompilerRunnerTest`, `SqljToolchainSettingsTest`).
  - New Project Wizard Tests (`SqljModuleBuilderTest`).
- [x] **Kompatibilitätstest & Verification**:
  - Plattform-Konfiguration validiert gegen Build `#261.24374-151` (`since-build="261.24374"`) und Java 21 LTS (`PluginConfigurationTest`).
- [x] **Marketplace Readiness**:
  - High-Quality Plugin Icons (`pluginIcon.svg`, `pluginIcon_dark.svg`, `pluginIcon-small.svg`, `sqlj-file.svg`).
  - Vollständige Plugin-Beschreibung, Dokumentation (`docs/`), Anleitungen & Web-Assets.
  - CI/CD Deployment-Konfigurationen für GitHub Actions und JetBrains Marketplace.

---

## 5. Lessons-Learned Checkliste (speziell für SQLJ)

Basierend auf den Erkenntnissen aus `docs/development/`:

1. **Unterscheide Koordinaten**:
   - Trenne strikt zwischen: SQLJ Quelltextbereich, injiziertem DB2 SQL Bereich, synthetischem Java Scaffolding und generiertem Java/Bytecode.
2. **Editor Recovery vs. Build**:
   - Unvollständiger SQLJ-Code (z.B. während des Tippens) darf nicht den gesamten Java-AST zerstören.
3. **Debugger Kontext-Trennung**:
   - Visuelle Breakpoint-Position liegt in der `.sqlj`-Datei; Expression Evaluation benötigt den Java PSI-Scope des Stackframes.
4. **Vermeide rekursive Extension-Point Aufrufe**:
   - In `PositionManager` und `BreakpointType` keine internen Accessors rekursiv aufrufen.
5. **Dumb Mode & Lifecycle**:
   - Alle PSI-Lookups und Index-Zugriffe müssen Dumb-Mode-resistent sein und EDT-Regeln strikt einhalten.
