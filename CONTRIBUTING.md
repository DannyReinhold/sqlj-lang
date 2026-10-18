<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Contributing to SQLJ Language Support (`sqlj-lang`)

Thank you for your interest in contributing to `sqlj-lang`! We welcome bug reports, feature suggestions, documentation improvements, and pull requests.

---

## Code of Conduct

Please maintain a respectful, welcoming, and collaborative tone in all discussions and pull requests.

---

## Maintainer & Contributor Documentation

For detailed, step-by-step guides on building, debugging, installing locally, and navigating the codebase, refer to our maintainer documentation:

- 📖 **[Maintainer Guide Index](docs/maintainers/index.md)**: Main hub for plugin contributors and maintainers.
- 🛠️ **[Building, Running & Local Installation](docs/maintainers/building-and-running.md)**: Detailed Gradle commands (`buildPlugin`, `runIde`, `test`), installing `.zip` archives from disk, and running in an isolated development sandbox.
- 🧭 **[Source Code & Architecture Walkthrough](docs/maintainers/source-code-walkthrough.md)**: Tour through all packages, AST/PSI models, SQL injection, synthetic classes, host variable binding, debugger position manager, and compiler integration.
- 🚀 **[Releasing & Deployment](docs/maintainers/releasing-and-deployment.md)**: Release checklists, versioning, signing, and JetBrains Marketplace publishing.

---

## Development Setup

1. **Clone the repository**:
   ```bash
   git clone ssh://git@git.rsas.de:2222/danny/sqlj-lang.git
   cd sqlj-lang
   ```

2. **Open in IntelliJ IDEA**:
   - Open the directory in IntelliJ IDEA.
   - Ensure the project SDK is configured for **Java 21**.
   - Sync Gradle project.

3. **Building & Running**:
   - Run tests: `./gradlew test` (or `.\gradlew.bat test` on Windows)
   - Run sandbox IDE: `./gradlew runIde` (or `.\gradlew.bat runIde` on Windows)
   - Build installable plugin archive: `./gradlew buildPlugin` (or `.\gradlew.bat buildPlugin` on Windows)
   - Run Plugin Verifier: `./gradlew runPluginVerifier`

---

## Coding Standards

- **Target Compatibility**: All code must run on Java 21 LTS and be compatible with IntelliJ Platform `#261.24374-151`+.
- **Architecture Guidelines**: Refer to [Source Code & Architecture Walkthrough](docs/maintainers/source-code-walkthrough.md) and `docs/development/` for crucial patterns regarding Multi-Host Injections, Reference Providers, and Debugger Position Managers.
- **Dumb Mode & Threading**: Respect IntelliJ threading conventions (read actions, write actions, non-blocking EDT operations, dumb mode checks).
- **Tests**: Write unit and integration tests for every new feature or bug fix.

---

## Submitting Pull Requests

1. With authorized access to the Gitea source repository, fork the repository and create a feature branch (`git checkout -b feature/my-feature`).
2. Commit your changes with descriptive commit messages.
3. Ensure all tests pass (`./gradlew test`).
4. Submit a Pull Request targeting the `main` branch.

## Licensing contributions

Project code is PolyForm Noncommercial 1.0.0; documentation is CC BY 4.0 and its
embedded code examples are MIT. See [Licensing](docs/licensing.md). Contributions intended for inclusion
in commercially licensed distributions also require permission for Danny Reinhold
to distribute them under those commercial terms. Arrange that permission before
submitting code; do not assume a noncommercial-only submission authorizes commercial
relicensing. Third-party code keeps its original license and attribution.
