<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->
# SQLJ Language Support — public documentation

Documentation for the independent SQLJ IntelliJ IDEA plugin by Danny Reinhold.


> [!WARNING]
> **FIRST DB2 START: ALLOW MANY MINUTES — PLEASE DO NOT ABORT EARLY.**
> Downloading the image and creating SAMPLE can take a very long time. The starter
> allows **15 minutes for readiness after container startup**, plus image download
> and build time. **The complete first launch can exceed 15 minutes.** A running
> container is not yet a ready database. Keep the build running and use
> `docker compose logs --follow db2` in a second terminal to see initialization.
> Later starts normally reuse the database and become ready much sooner.

- [Website and overview](https://sqlj-lang.org)
- [Documentation index](docs/documentation-map.md)
- [Getting started](docs/user-guide/getting-started.md)
- [Debugging Java, Spring and WebSphere Liberty](docs/user-guide/debugging-project-types.md)
- [Licensing](docs/licensing.md)
- [Contributor information](CONTRIBUTING.md)

This repository contains documentation and website assets only. Plugin source,
implementation tests, standalone example source files, binaries and build history
are maintained separately on Gitea and are not mirrored here. Changes here are
replaced by the documentation publisher; report documentation fixes to the maintainer.

Documentation and project-owned website artwork: Copyright 2026 Danny Reinhold,
licensed under [CC BY 4.0](LICENSE). Embedded documentation code examples are
additionally available under [MIT](LICENSE-CODE-EXAMPLES). Third-party material
retains its own terms; trademarks are not licensed by these grants.

The plugin itself is PolyForm Noncommercial 1.0.0, with a separate commercial
licensing alternative. This repository's documentation license does not license
the plugin for commercial use.
