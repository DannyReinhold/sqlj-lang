<!-- SPDX-License-Identifier: CC-BY-4.0; code examples: MIT -->

# Integrate debug metadata into an existing SQLJ build

Use your existing application, translator, binding process and deployment. You do
not need to generate a wizard application, copy its POM or adopt Spring Boot.
This guide is for the people maintaining that build; developers then use the
[Java/Spring/Liberty debugger setup](debugging-project-types.md).

The SQLJ plugin uses source maps embedded in the classes actually loaded by the
JVM. If your build already embeds matching SQLJ SMAP metadata, retain that process;
the supplied tool is an alternative installer, not an additional mandatory stage.
Do not install two different mappings into the same class.

## Obtain and pin the tool without generating a project

The plugin distribution contains an executable `wizard/sqlj-debug-tool.jar` resource
inside its plugin JAR. It is supplied under MIT, including the license inside the
build JAR. Obtain the plugin ZIP through your approved distribution channel.
An administrator can extract the tool once with Python 3's standard library:

```python
# Save as extract_sqlj_build_tool.py; run once, outside the application's build.
from pathlib import Path
from io import BytesIO
from zipfile import ZipFile
import sys

archive, destination = map(Path, sys.argv[1:])
found = []
with ZipFile(archive) as distribution:
    for name in distribution.namelist():
        if not name.endswith('.jar'):
            continue
        with ZipFile(BytesIO(distribution.read(name))) as plugin:
            if 'wizard/sqlj-debug-tool.jar' in plugin.namelist():
                found.append(plugin.read('wizard/sqlj-debug-tool.jar'))
if len(found) != 1:
    raise SystemExit('Expected exactly one SQLJ debug tool in the plugin ZIP')
# Refuse to overwrite a pinned version accidentally.
destination.parent.mkdir(parents=True, exist_ok=True)
with destination.open('xb') as output:
    output.write(found[0])
print(destination)
```

For example, from the existing application's root in PowerShell:

```powershell
py -3 .\extract_sqlj_build_tool.py C:\Downloads\sqlj-lang-1.0.0.zip .\.mvn\sqlj\sqlj-debug-tool.jar
Get-FileHash .\.mvn\sqlj\sqlj-debug-tool.jar -Algorithm SHA256
```

Substitute the actual approved ZIP name. Commit the tool with the application or
store that exact binary in your internal artifact repository, with its hash,
plugin distribution version and MIT notice. There is no public Maven coordinate
to resolve for this tool. Python is only used for this one-time extraction; it is
not needed during the application's build. IDE installation is not needed on CI.
A plugin update does not update pinned application build tools automatically.

The supplied tool runs on **JDK 21 or newer**, separately from the application's
compiler/runtime. Keep the existing application release target and server JVM.
If Maven itself runs on an older JDK, point `sqlj.debug.java` below at a JDK 21
`java` executable. This does not change the compiler toolchain. Verify your target
class-file format and actual translator output when adopting the tool.

## Build sequence and command-line contract

Run these stages per module, with separate source/generated/class directories and
a separate manifest for each module:

1. **prepare** snapshots the UTF-8 `.sqlj` inputs and removes old `.java.smap`
   sidecars from the configured generated-source tree. Run it before translation.
2. Your existing translator emits generated Java and the corresponding
   `.java.smap` files. For the wizard's IBM translator the options are
   `-smap=yes -linemap=no -compile=false`. Confirm equivalent options with your
   actual translator; do not silently substitute a different vendor tool.
3. Your existing Java compiler emits classes with `-g:lines,vars,source`.
4. **install** validates maps, sources and matching compiled classes, then embeds
   their SMAP and source fingerprints, including matching nested classes.
5. Package, sign and deploy the resulting classes using your existing process.

Both modes take exactly the same four paths, in this order:

```text
java -jar sqlj-debug-tool.jar prepare sourceRoot generatedRoot classesRoot manifest
java -jar sqlj-debug-tool.jar install sourceRoot generatedRoot classesRoot manifest
```

`sourceRoot` is the SQLJ source root (packages below it); `generatedRoot` contains
generated Java and its adjacent `.java.smap` files; `classesRoot` is the compiler
output directory. `manifest` is a build-local file written by prepare and read by
install. Paths may be absolute or relative to the command's working directory.
Quote each path containing spaces. Missing maps, changed sources and inconsistent
classes fail the command with a nonzero exit code; make that fail the build.
Never run prepare after translation: it intentionally removes sidecars.

Limit each invocation to the module's translated SQLJ inputs. Prepare recursively
finds `.sqlj` sources, but **does not translate them**; extend your existing
translator input list explicitly. It does not verify that every snapshotted input
produced a map. Keep your translator's own input/output completeness checks.
The installer accepts one SQLJ input and one stratum per map and requires matching
package/SourceFile class metadata. It is not an adapter for arbitrary proprietary
map formats. Keep binding/profile customization unchanged and preserve all SQLJ
runtime/profile resources when packaging.

## Maven integration fragments

Merge these fragments into the appropriate existing module; do not replace its
POM or add a second compiler/exec plugin declaration. Use an existing approved
exec-plugin version if already managed by your organization.

```xml
<properties>
    <sqlj.debug.java>${java.home}/bin/java</sqlj.debug.java>
    <sqlj.debug.tool>${project.basedir}/.mvn/sqlj/sqlj-debug-tool.jar</sqlj.debug.tool>
    <sqlj.debug.sources>${project.basedir}/src/main/sqlj</sqlj.debug.sources>
    <sqlj.debug.generated>${project.build.directory}/generated-sources/sqlj</sqlj.debug.generated>
    <sqlj.debug.manifest>${project.build.directory}/sqlj-debug-inputs.properties</sqlj.debug.manifest>
</properties>
```

For multiple modules, either supply a pinned JAR to each module or configure
`sqlj.debug.tool` to the shared approved location explicitly. Do not assume a
module's `project.basedir` is the reactor root.

Inside the existing `build/plugins`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <debug>true</debug>
        <debuglevel>lines,vars,source</debuglevel>
    </configuration>
</plugin>
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <version>3.5.0</version>
    <executions>
        <execution>
            <id>sqlj-debug-prepare</id>
            <phase>initialize</phase>
            <goals><goal>exec</goal></goals>
            <configuration>
                <executable>${sqlj.debug.java}</executable>
                <workingDirectory>${project.basedir}</workingDirectory>
                <arguments>
                    <argument>-jar</argument><argument>${sqlj.debug.tool}</argument>
                    <argument>prepare</argument>
                    <argument>${sqlj.debug.sources}</argument>
                    <argument>${sqlj.debug.generated}</argument>
                    <argument>${project.build.outputDirectory}</argument>
                    <argument>${sqlj.debug.manifest}</argument>
                </arguments>
            </configuration>
        </execution>
        <execution>
            <id>sqlj-debug-install</id>
            <phase>process-classes</phase>
            <goals><goal>exec</goal></goals>
            <configuration>
                <executable>${sqlj.debug.java}</executable>
                <workingDirectory>${project.basedir}</workingDirectory>
                <arguments>
                    <argument>-jar</argument><argument>${sqlj.debug.tool}</argument>
                    <argument>install</argument>
                    <argument>${sqlj.debug.sources}</argument>
                    <argument>${sqlj.debug.generated}</argument>
                    <argument>${project.build.outputDirectory}</argument>
                    <argument>${sqlj.debug.manifest}</argument>
                </arguments>
            </configuration>
        </execution>
    </executions>
</plugin>
```

This assumes translation runs after initialize (usually generate-sources) and
Java compilation finishes before process-classes. Adapt the phases/order to the
real build, including inherited plugins and active profiles. Register generated
Java with the compiler using the project's existing mechanism. Running
`mvn compile` alone does not reach installation; use `mvn process-classes` for a
local class-directory launch or `mvn package` / the normal deployment goals.
These fragments intentionally do not configure your translator, database,
credentials, binding or application server.

Run installation after any stage that recompiles/replaces classes and before
packaging/signing. Later transformations must preserve source maps and line tables.
For Ant, Gradle or scripts, invoke the same two commands at the equivalent points
and propagate failures. Keep build-only tools out of the application runtime
classpath and WAR/JAR dependencies.

## IDE, CI and deployment hand-off

CI must use the same pinned tool and source revision as the developer build.
A terminal build must install maps too; IDE hooks alone cannot fix CI artifacts.
For ordinary Java/Spring launches, configure **SQLJ: Prepare Debug Build** to run
the appropriate Maven goals. That task runs your build; it does not add these
executions to a POM. Put it after other build tasks that would overwrite classes.
For Liberty or another remote server, package and deploy the mapped classes,
then attach to that JVM; a local process-classes build does not redeploy a server.

Check the resulting class with `javap -v`: it must contain `SourceDebugExtension`
and a SQLJ stratum. Also inspect the deployed artifact, not just local target
output. Stop/redeploy/restart as required by the server, then verify a source
breakpoint on an executable SQLJ line. The plugin's **SQLJ: Inspect Debug Session**
action can inspect loaded metadata at a paused frame. See the
[debugger setup and diagnostics](debugging-project-types.md) for details.
