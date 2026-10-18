<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Building, Running & Installing the SQLJ Plugin Locally

This guide details all necessary commands and workflows for building the SQLJ plugin, running it inside an isolated development sandbox, and creating installable `.zip` distribution packages for manual verification in local IDE installations.

---

## Prerequisites

- **Java Development Kit (JDK)**: JDK 21 LTS or higher configured as `JAVA_HOME`.
- **Operating System**: Windows, macOS, or Linux.
- **Git**: Configured with line endings matching your platform.

---

## 1. Running Unit & Integration Tests

Before building packages or committing changes, verify that all test suites pass without regression:

### Linux / macOS
```bash
./gradlew test
```

### Windows (PowerShell / Command Prompt)
```powershell
.\gradlew.bat test
```

### Running a Specific Test Class or Method
To run targeted tests during rapid development loops:

```bash
# Run single test class
./gradlew test --tests org.sqlj.lang.wizard.SqljModuleBuilderTest

# Run single test method
./gradlew test --tests org.sqlj.lang.wizard.SqljModuleBuilderTest.testGenerateProjectFiles

# Force clean test rerun
./gradlew test --rerun-tasks
```

---

## 2. Launching the Sandbox Development IDE (`runIde`)

The fastest and most efficient way to test plugin features interactively (without installing archives manually) is to spawn an isolated IntelliJ IDEA sandbox instance with the plugin automatically deployed:

### Linux / macOS
```bash
./gradlew runIde
```

### Windows (PowerShell / Command Prompt)
```powershell
.\gradlew.bat runIde
```

### Advantages of `runIde`:
- **Isolated Sandbox**: Runs with a separate sandbox configuration and plugin directory under `build/idea-sandbox/`. Your personal IDE settings and installed plugins remain completely untouched.
- **Immediate Testing**: You can immediately test:
  - File associations (`.sqlj` file icons and syntax highlighting)
  - Code completion inside `#sql` statements and Java host expressions
  - The **New Project Wizard** (`File` -> `New` -> `Project...` -> `SQLJ Spring Boot Application`)
  - SQLJ toolchain settings page under `Settings` -> `Languages & Frameworks` -> `SQLJ & DB2 Toolchain`
  - Breakpoint placement and debugging in `.sqlj` sources
- **Hot Debugging**: If you run `./gradlew runIde` from within IntelliJ IDEA's Gradle tool window with the "Debug" button, IntelliJ connects a debugger, allowing you to set breakpoints inside plugin classes themselves.

---

## 3. Building the Installable Plugin Archive (`buildPlugin`)

To distribute the plugin or install it into a production/local IntelliJ IDEA installation, build the distribution ZIP:

### Linux / macOS
```bash
./gradlew buildPlugin
```

### Windows (PowerShell / Command Prompt)
```powershell
.\gradlew.bat buildPlugin
```

### Performing a Clean Build
When switching branches, updating dependencies, or ensuring no stale build cache artifacts exist:

```bash
# Linux / macOS
./gradlew clean buildPlugin

# Windows PowerShell
.\gradlew.bat clean buildPlugin
```

### Build Artifact Location
Upon successful execution, the packaged plugin ZIP file is placed in:
```
build/distributions/sqlj-lang-<version>.zip
```
*(e.g., `build/distributions/sqlj-lang-0.1.0-SNAPSHOT.zip`)*

---

## 4. Manual Installation in IntelliJ IDEA from Disk

Once the `.zip` archive has been generated via `buildPlugin`, follow these steps to install and test it in your IDE:

1. Open **IntelliJ IDEA**.
2. Open **Settings** (Windows/Linux: `Ctrl+Alt+S` | macOS: `Cmd+,`).
3. Select **Plugins** in the left navigation pane.
4. Click the gear icon (⚙) at the top of the Plugins panel (next to the *Installed* tab).
5. Choose **Install Plugin from Disk...** from the dropdown menu.
6. In the file chooser, navigate to your repository clone and select:
   `build/distributions/sqlj-lang-<version>.zip`
7. Click **OK** / **Apply**.
8. Restart IntelliJ IDEA when prompted.

### Verifying Successful Installation
- Check **Settings** -> **Plugins** -> **Installed**: Verify that **SQLJ Language Support** is listed and enabled.
- Verify that opening a `.sqlj` file displays the SQLJ icon and provides syntax highlighting.
- Check **Settings** -> **Languages & Frameworks** -> **SQLJ & DB2 Toolchain** to confirm settings registration.

---

## 5. Running the IntelliJ Plugin Verifier

The IntelliJ Plugin Verifier checks plugin binaries against targeted IntelliJ Platform releases to detect binary incompatibilities, deprecated API usages, and missing dependencies:

### Linux / macOS
```bash
./gradlew runPluginVerifier
```

### Windows (PowerShell)
```powershell
.\gradlew.bat runPluginVerifier
```

The verification report will be generated under `build/reports/pluginVerifier/`.

---

## 6. Summary of Key Gradle Tasks

| Gradle Task | Purpose |
|---|---|
| `test` | Compiles and executes all unit and integration tests. |
| `runIde` | Launches a sandboxed IntelliJ instance with the plugin active. |
| `buildPlugin` | Packages the plugin into a distributable `.zip` file in `build/distributions/`. |
| `clean` | Cleans the `build/` directory and build caches. |
| `runPluginVerifier` | Validates binary compatibility against configured IDE versions. |
| `verifyPlugin` | Validates plugin descriptor (`plugin.xml`) structure and extensions. |
| `patchPluginXml` | Injects version numbers and change notes into `plugin.xml`. |

---

## Next Steps
- Continue with the [Source Code & Architecture Walkthrough](source-code-walkthrough.md) to understand how the plugin is structured internally.
- Check [Releasing & Deployment](releasing-and-deployment.md) for publishing workflows.
