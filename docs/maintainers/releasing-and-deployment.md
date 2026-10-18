<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Releasing & Deployment Guide

This document outlines the release process, versioning policy, and deployment workflow for the **SQLJ Language Support Plugin (`sqlj-lang`)**.

---

## 1. Release Checklist

Before creating a release, complete the following pre-flight verifications:

0. **Licensing and publication scope**:
   - Review LICENSE/NOTICE and packaged licensing files. PolyForm Noncommercial
     has no Change Date; the former BSL date reminder no longer applies.
   - Preserve third-party notices and previously granted rights for older copies.
   - Run `python3 -m unittest discover -s scripts/tests -v` and
     `python3 scripts/export_public_docs.py` (Windows: `py -3` instead of `python3`).
   - Publish source/releases only from Gitea. GitHub receives documentation only,
     using .gitea/workflows/publish-docs.yml and the explicit export manifest.
   - Never upload plugin ZIPs, source archives or code branches/tags to GitHub.

1. **Test Suite Verification**:
   ```bash
   ./gradlew clean test --rerun-tasks
   ```
   Ensure 100% of unit, PSI, injection, debugger, and wizard tests pass.

2. **Plugin Verifier Check**:
   ```bash
   ./gradlew runPluginVerifier
   ```
   Confirm zero binary incompatibilities or critical warnings against targeted IntelliJ Platform builds.

3. **Descriptor & Metadata Verification**:
   ```bash
   ./gradlew verifyPlugin
   ```
   Ensure `plugin.xml` declarations, vendor tags, and change notes are valid.

4. **Sandbox Smoke Test**:
   ```bash
   ./gradlew runIde
   ```
   Verify:
   - Creating a new project via the Wizard (`SQLJ Spring Boot Application`).
   - Editing and syntax highlighting of `.sqlj` files.
   - Code completion and navigation (`Ctrl+B`).

---

## 2. Versioning Policy

The project adheres to [Semantic Versioning (SemVer)](https://semver.org/):

- **Major (`X.0.0`)**: Incompatible platform migrations or major architectural overhauls.
- **Minor (`0.X.0`)**: New IDE features, language capabilities, or toolchain support.
- **Patch (`0.0.X`)**: Bug fixes, performance improvements, and diagnostic updates.

Update the version number in `gradle.properties`:
```properties
pluginVersion = 1.0.0
```

---

## 3. Building Release Artifacts

Generate the release distribution ZIP package:

```bash
# Linux / macOS
./gradlew clean buildPlugin

# Windows PowerShell
.\gradlew.bat clean buildPlugin
```

The resulting package will be stored in:
`build/distributions/sqlj-lang-<version>.zip`

---

## 4. Publishing to JetBrains Marketplace

### Automated CI/CD Publishing
When Gitea Actions is enabled, the private .gitea/workflows/release.yml workflow
can publish Marketplace artifacts on a published Gitea release. Configure these
repository secrets (not on the public GitHub documentation repository):
- `JB_MARKETPLACE_TOKEN`
- `JB_CERTIFICATE_CHAIN`
- `JB_PRIVATE_KEY`
- `JB_PRIVATE_KEY_PASSWORD`

### Manual Publishing
To publish manually via Gradle:
```bash
./gradlew publishPlugin
```

Or upload `build/distributions/sqlj-lang-<version>.zip` directly to the [JetBrains Marketplace Plugin Dashboard](https://plugins.jetbrains.com/).

Repository migration and documentation deployment setup are described in MIGRATION.md
in the canonical source repository. Publication must first pass the documented dry-run.
