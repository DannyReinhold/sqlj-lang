<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Administration: approve, distribute and maintain the plugin

This guide is for platform teams, IDE administrators and software distribution
staff. Developers should start with [getting started](../user-guide/getting-started.md).
The source repository is private. Administrators install an approved binary;
they do not need source access, Git credentials, a compiler or a DB2 container
just to distribute the plugin.

## 1. Establish the approved configuration

Record the plugin version/archive SHA-256, exact IntelliJ build and edition,
Java plugin status, optional Maven and Database Tools availability and permitted
usage. The [compatibility matrix](../user-guide/compatibility.md) distinguishes
observed behavior from unverified targets. The manifest's minimum build alone
is not a tested compatibility matrix. Other JetBrains IDEs are not promised.

The IDE's own runtime, the application's JDK and the Maven runner JDK are separate.
For the generated demo, application/Maven Java must be JDK 21. Existing
applications retain their approved compiler/runtime requirements. Database Tools
features can require a separate JetBrains entitlement.

Review [licensing](../licensing.md) before rollout. The documentation's CC BY license
and the generated starter's MIT license do not license the plugin for commercial
use. Obtain appropriate commercial terms from the maintainer when required; do
not infer entitlement from the ability to install or copy a ZIP. No central
license-server integration or automatic commercial-license enforcement is promised.

## 2. Obtain and distribute the artifact

Obtain the exact ZIP and release information from the maintainer/approved internal
software channel. The GitHub documentation repository has no plugin binaries or
source releases. Use Marketplace only when the applicable listing/version is
actually available. Preserve packaged license and third-party notices.

Compute the received archive's hash and compare it with a value independently
supplied by your trusted distributor when available. Computing a hash alone does
not authenticate a download:

```powershell
Get-FileHash -Algorithm SHA256 .\sqlj-lang-<version>.zip
```

Replace `<version>` with the actual delivered version. Keep the approved archive,
its hash and release notes for rollback. For a pilot, install through **Settings →
Plugins → gear menu → Install Plugin from Disk**, then restart as requested.
Confirm the installed plugin's name/version and enabled Java dependency.

For centrally managed IDEs, use the organization's existing approved plugin
repository/distribution mechanism and installation policy. This project does not
provide an enterprise update server or fleet-management service. JetBrains
[plugin-management documentation](https://www.jetbrains.com/help/idea/managing-plugins.html)
describes disk installation, versions and custom repositories; availability can
be restricted by the organization's IDE policy. Do not advise developers to
bypass such restrictions.

## 3. Pilot before broad rollout

Use a disposable development workspace and a representative application:

1. Confirm `.sqlj` file recognition without the optional database plugin.
2. In an imported project, verify source roots, first-line Java-view marker and
   SQLJ runtime dependencies; test a valid reference and an intentionally wrong
   Java method/import. A file looking normal is not an adequate error-check test.
3. If licensed/enabled, verify DB2 SQL completion against a development data source.
4. If debugging is required, build fresh maps and test a real SQLJ breakpoint.
   Record stepping/evaluation separately using the [acceptance checks](../user-guide/debugging-checks.md).
5. Confirm normal Java editing/debugging still behaves correctly. Test your exact
   Maven profiles and build/deploy pipeline; the wizard does not certify them.

The demo requires separate approval for Docker/image downloads and a privileged
DB2 container. It exposes database port 50000 and HTTP port 8080, uses demo
credentials and persistent database storage. Review port binding, volumes and
IBM image/tool licensing. Do not deploy that configuration to production. DB2
containers and first-start waits are demo requirements, not requirements for
installing the plugin or editing an existing enterprise project.

## 4. Configuration and data handling

| Item | Handling |
| --- | --- |
| Plugin Toolchain settings | Project state in `.idea/sqlj.xml`; the password is an ordinary persisted field, not Password Safe. Do not store production secrets there or commit the file with credentials. |
| Maven preparation | Per-run-configuration POM/goals. It saves editor documents and executes the selected imported Maven project's build. Review shared run configurations and goals as executable project configuration. |
| Runtime/source data | Editor features inspect project files. Debug diagnostics inspect paused JVM metadata and variable names/types, not variable values. The user must explicitly copy/share reports. |
| Network/build access | Plugin tasks delegate to Maven; the demo invokes Docker and downloads dependencies/images. Optional IDE database tooling can connect to configured data sources. No blanket offline or no-network guarantee. |
| Shared diagnostics | Navigation reports and IDE logs can contain paths, packages, names and source excerpts. Redact before sharing. Debug expression evaluation is native Java and can invoke application methods when the user requests it. |

The plugin does not define a central credential vault or server access policy.
The [website privacy notice](../privacy.md) concerns website visitors; it is not a
claim that local Maven, Docker, IDE database services or user evaluation are free
of data processing. Follow your existing development-security controls.

## 5. Update, rollback and removal

Pilot an update against the same acceptance checks, then distribute the approved
version through your usual channel. Do not assume every later IDE build is
compatible simply because the package has no upper-build limit.

For a regression, record the affected versions and reproduce in a development
workspace. Disable **SQLJ Language Support** under **Settings → Plugins → Installed**
and restart if needed. To roll back, retain/install the previous approved ZIP
through disk installation and verify the displayed version after restart. Removal
uses the same Installed page. Do not delete project sources or generated databases.

Disabling/removing the plugin does not revert a project's POM, SQLJ marker,
`.run` files, toolchain settings or Docker resources. Remove/disable the plugin's
before-launch task if you need to launch that project without the plugin, and
use the application's established build process. Container cleanup is a separate
operation under the project's data-retention policy.

## Handoff to developers and support

Supply the approved version, installation channel, supported IDE combination,
license contact, required project configuration, internal Maven/image access and
support route. Developers own their application source/build/deployment settings;
platform teams own approval/distribution. The [support checklist](../user-guide/troubleshooting.md)
helps distinguish IDE setup, plugin behavior, translator failures and runtime issues.
