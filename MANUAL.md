Documentation
-------------

* [Using the SuperPOM](#using-the-superpom)
* [What the default build does](#what-the-default-build-does)
* [Compiler](#compiler)
* [Tests](#tests)
* [Code quality tools](#code-quality-tools)
* [Javadoc](#javadoc)
* [License headers](#license-headers)
* [Profiles](#profiles)
* [Continuous Integration, metrics and site](#continuous-integration-metrics-and-site)
* [The release cycle](#the-release-cycle)
* [Assemblies and extra artifacts](#assemblies-and-extra-artifacts)
* [Web applications](#web-applications)
* [AspectJ and Spring AOP](#aspectj-and-spring-aop)
* [JavaFX and macOS application bundles](#javafx-and-macos-application-bundles)
* [Debian packages](#debian-packages)
* [Spring Boot and GraalVM native images](#spring-boot-and-graalvm-native-images)
* [Maintenance utilities](#maintenance-utilities)
* [Automatically activated profiles](#automatically-activated-profiles)
* [Properties reference](#properties-reference)
* [Managed plugin versions](#managed-plugin-versions)
* [Properties you must provide](#properties-you-must-provide)
* [Deprecated and unused properties](#deprecated-and-unused-properties)


### Using the SuperPOM

Declare it as the parent of your project:

```xml
<parent>
    <groupId>it.tidalwave.thesefoolishthings</groupId>
    <artifactId>superpom</artifactId>
    <version>-- version --</version>
    <relativePath />
</parent>
```

Then override the project metadata (see [Remember to customise it](#remember-to-customise-it)) and the SCM
coordinates, which are driven by properties:

| Property                    | Used for                                                                               |
|-----------------------------|----------------------------------------------------------------------------------------|
| `scm.repo.url`              | Read/write repository URL (`<developerConnection>`, through `scm.repo.effective.url`). |
| `scm.repo.readonly.url`     | Read-only repository URL (`<connection>`).                                             |
| `scm.repo.browse.url`       | Web URL for browsing the sources (`<url>`).                                            |
| `scm.repo.effective.url`    | Defaults to `scm.repo.url`; override it to push to a different location.               |
| `tft.scm`                   | SCM provider used in the connection strings (default `git`).                           |

Almost every plugin setting is exposed as a `tft.*` property, so customising the build usually means setting a property
in your `<properties>` section (or with `-D` on the command line) instead of copying a plugin section.


### What the default build does

* **Default goal**: running `mvn` with no goal executes `clean install` (property `tft.defaultGoal`).
* **Requirements**: the Maven Enforcer Plugin requires Maven `3.9.0` or later (`tft.maven.requiredVersion`) and a JDK in
  the range `[11,)` (`tft.maven.requiredJdk`). It also requires every plugin to have an explicit version (no `LATEST`,
  `RELEASE` or `SNAPSHOT`, checked for the `clean`, `deploy` and `site` phases) and checks reactor module convergence.
* **Reproducible plugin versions**: every plugin listed under [Managed plugin versions](#managed-plugin-versions) has its
  version pinned in `<pluginManagement>` through a `tft.<plugin>.version` property, which you can override.
* **Build number**: the Build Number Plugin runs at `validate` and stores the current SCM revision (12 characters, or
  `unknown` if the SCM can't be queried) in `${buildNumber}`.
* **Final name**: artifacts are named `${project.artifactId}-${project.version}.${buildNumber}`, so every build is
  traceable to a commit. The `release` and `it.tidalwave-deploy-extra-artifacts-v2` profiles drop the build number.
* **JAR manifest**: default implementation and specification entries are added, plus `Implementation-Title` and
  `Implementation-Version` (`${project.version}.${buildNumber}`). Set `tft.jar.classpathInManifest` to `true` to add the
  `Class-Path` entry.
* **Test JAR**: a `test-jar` artifact is produced at `package` whenever there are test classes.
* **Encoding**: sources are UTF-8 (`project.build.sourceEncoding`), also used for the site.
* **Deploy**: deployment can be disabled with `tft.disableDeploy=true`. The default distribution repository is the
  Maven Central Portal (`tft.staging-repo.url`, server id `central`).
* **Build extensions**: `wagon-webdav-jackrabbit` (WebDAV deployment via `dav:` URLs) and `jdeb` (enables `deb`
  packaging).
* **Run**: `mvn exec:java` launches the class named in `tft.mainClass`.


### Compiler

The Compiler Plugin is configured with:

* `-source` / `-target` from `tft.javac.source` and `tft.javac.target` (default `11`); on JDK 9 and later
  (`javac-jdk9plus` profile) also `--release` from `tft.javac.release` (default `11`);
* `-Xlint:-processing` plus `tft.javac.xlint` (default `-Xlint:all`);
* debug information, deprecation and warning output (`tft.javac.debug`, `tft.javac.showDeprecation`,
  `tft.javac.showWarnings`, all `true`);
* a forked compiler (`tft.javac.fork`, default `true`);
* the module path for test compilation, controlled by `tft.modulePathInTests` (default `true`).


### Tests

Tests run with Surefire using this configuration:

* only classes matching `**/*Test.java` are run, in alphabetical order;
* the JVM is started with `-Xmx${tft.test.maxMemory}` (default `768M`), plus `tft.test.vmOptions` and
  `tft.test.allowAttachSelf` (`-Djdk.attach.allowAttachSelf=true`; on JDK 21 and later also
  `-XX:+EnableDynamicAgentLoading`, which silences the dynamic agent warnings raised by mocking libraries);
* test groups listed in `tft.test.excludedGroups` are excluded;
* reports go to `target/surefire-reports/plain` (`tft.test.reportsDirectory`);
* tests are skipped with `-DskipTests` or `tft.test.skip=true`; failures are ignored with `tft.test.failureIgnore=true`;
* the module path is used when `tft.modulePathInTests` is `true`.

For headless JavaFX tests, activate `it.tidalwave-monocle-profile-v1` (with `-P`): it adds
`org.testfx:openjfx-monocle` (`tft.openjfx-monocle.version`, default `11.0.2`) as a test dependency and sets
`tft.test.vmOptions` to `-Dtestfx.robot=glass -Dtestfx.headless=true -Dprism.order=sw`.


### Code quality tools

These tools are configured in the main build section (not in a profile), so IDEs pick up the same settings. Their
configuration is replicated in the reporting section used by the site.

| Tool                       | Configuration                                                                                                                                                                                                                                                                                                                                           | Disable with |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
| **SpotBugs**               | Effort `Max`, threshold `Low`, XML output, max heap 512 MB, 0 allowed violations, doesn't fail the build (`tft.spotbugs.*`).                                                                                                                                                                                                                            | `tft.spotbugs.skip=true` (defaults to the value of `tft.findbugs.skip`) |
| **Checkstyle**             | Rules from `it/tidalwave/checkstyle/checkstyle.xml` in `it.tidalwave.thesefoolishthings:superpom-config` (`tft.superpom.config.version`).                                                                                                                                                                                                               | `tft.checkstyle.skip=true` |
| **PMD / CPD**              | Ruleset `tft.pmd.ruleset` (default `/it/tidalwave/pmd/standard.xml`) loaded from the artifact set by `tft.pmd.ruleset.groupId`, `tft.pmd.ruleset.artifactId` and `tft.pmd.ruleset.version` (defaults to `superpom-config`). CPD threshold `tft.pmd.minimumTokens` (100), minimum priority `tft.pmd.minimumPriority` (5), target JDK `tft.javac.source`. | `tft.pmd.skip=true` (also sets `pmd.skip` and `cpd.skip`) |
| **OWASP Dependency-Check** | XML and HTML reports, .NET assembly and MSBuild analyzers disabled.                                                                                                                                                                                                                                                                                     | `tft.owasp.skip=true` |
| **RevAPI**                 | API compatibility check (`revapi:check`, execution `api-check`) with `revapi-java` (`tft.revapi-java.version`).                                                                                                                                                                                                                                         | Skipped by default; enable with `tft.revapi.skip=false` |

None of these tools runs during a plain `mvn install`. They are run by the metrics and site profiles described below,
or can be invoked directly (e.g. `mvn spotbugs:check`).


### Javadoc

* Packages matching `tft.javadoc.excludePackageNames` (default `*.impl:*.impl.*`) are excluded;
* `doc-files` subdirectories are copied, errors don't fail the build, output is quiet (`tft.javadoc.quiet`);
* `tft.javadoc.legacyMode` (default `false`) and `tft.javadoc.detectJavaApiLink` (default `false`) are exposed;
* on JDK 9 and later (`umldoclet-jdk9plus` profile) the [UML Doclet](https://github.com/talsma-ict/umldoclet) is used
  (`tft.javadoc.doclet`, `tft.javadoc.doclet.groupId`, `tft.javadoc.doclet.artifactId`,
  `tft.javadoc.doclet.version` = `2.2.2`), with extra options from `tft.javadoc.doclet.additionalParam` and
  `tft.javadoc.doclint` (e.g. set it to `-Xdoclint:none`);
* Javadoc generation can be disabled with `tft.disableJavaDocs=true`.


### License headers

The Codehaus License Maven Plugin is configured to manage file headers under `src`:

* the license is `tft.license.name` (default `tidalwave_asf2`), resolved from `classpath://it/tidalwave/license` in
  `superpom-config`;
* the header template is `tft.license.template`;
* copyright years and descriptions can be updated;
* `LICENSE.txt`, SCM ignore files, IDE files, `*.form`, `*.mf`, `.description`, archetypes and `*-profile` marker files
  are excluded.

Use the profiles `it.tidalwave-update-headers` and `it.tidalwave-check-headers` (see
[Maintenance utilities](#maintenance-utilities)) to apply or verify the headers.


### Profiles

Many profiles are activated by the presence of an **empty marker file** in the module, usually under `src/config`. This
lets a project opt into a feature without touching its POM, e.g.:

```shell
mkdir -p src/config
touch src/config/activate-it.tidalwave-jetty-v2-profile
```

Other profiles are activated explicitly with `-P<id>`, by the JDK version, or by a property. The table below summarises
all of them; the following sections describe each one in detail.

| Profile                                          | Activation                                                                           | Purpose                                                             |
|--------------------------------------------------|--------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| `it.tidalwave-ci-v1`                             | `-P`                                                                                 | CI build: exclude `no-ci` tests, ignore test failures               |
| `it.tidalwave-metrics-v2`                        | `-P`                                                                                 | Full QA run with JaCoCo, SpotBugs, PMD, Checkstyle, Sonar, OWASP    |
| `it.tidalwave-metrics-pre-site-v1`               | `-P`                                                                                 | Skip in metrics the tools that the site will run again              |
| `it.tidalwave-site-v1`                           | `-P`                                                                                 | Generate and stage the Maven site with all reports                  |
| `it.tidalwave-site-docs-maven-skin-v1`           | file `src/config/activate-it.tidalwave-site-docs-maven-skin-v1-profile`              | Use the Docs Maven Skin for the site                                |
| `it.tidalwave-jacoco-aggregate-v1`               | directory `modules` exists                                                           | JaCoCo report aggregating all the modules, in the site of the root  |
| `generate-revision-id`                           | `-P`                                                                                 | Write the build description to `.description`                       |
| `show-revision-id`                               | `-P`                                                                                 | Print the build description stored in `.description`                |
| `release`                                        | `-P`                                                                                 | Release build, staging artifacts in a local repository              |
| `release-sign-artifacts`                         | passed by the Release Plugin                                                         | Sign artifacts with GPG                                             |
| `release-commit`                                 | `-P`                                                                                 | Publish staged artifacts with Wagon and push SCM changes            |
| `release-commit-central`                         | `-P`                                                                                 | Upload the staged bundle to the Central Portal and push SCM changes |
| `release-cancel`                                 | `-P`                                                                                 | Discard a staged release (for Jenkins workspaces)                   |
| `scm-detector-git` / `scm-detector-hg`           | `.git` / `.hg` exists                                                                | Detect the SCM for the release profiles                             |
| `it.tidalwave-deploy-extra-artifacts-v2`         | `-P`                                                                                 | Build and deploy assembly artifacts                                 |
| `it.tidalwave-assembly-bin-descriptor`           | file `src/assemble/bin.xml`                                                          | `bin` tarball assembly                                              |
| `it.tidalwave-assembly-project-descriptor`       | file `src/assemble/project.xml`                                                      | `project` (sources) tarball assembly                                |
| `it.tidalwave-assembly-tar.gz-v1`                | file `src/main/assembly/tar.gz.xml`                                                  | tar.gz assembly at `package`                                        |
| `it.tidalwave-assembly-zip-v1`                   | file `src/main/assembly/zip.xml`                                                     | zip assembly at `package`                                           |
| `it.tidalwave-distribution-v1`                   | file `src/config/activate-it.tidalwave-distribution-v1-profile`                      | Prepare files for an installer in `target/dist`                     |
| `it.tidalwave-manifest-with-main-v1`             | file `src/config/it.tidalwave-manifest-with-main-v1`                                 | Put `Main-Class` in the JAR manifest                                |
| `tomcat`                                         | file `src/config/activate-tomcat-profile`                                            | Configure the Tomcat 7 plugin                                       |
| `it.tidalwave-jetty-v2`                          | file `src/config/activate-it.tidalwave-jetty-v2-profile`                             | Configure the Jetty plugin                                          |
| `it.tidalwave-webresources-v1`                   | file `src/config/activate-it.tidalwave-webresources-v1-profile`                      | Package `src/main/webapp` as resources                              |
| `vaadin-compile-widgetset`                       | `-P`                                                                                 | Compile Vaadin widgetsets with GWT                                  |
| `it.tidalwave-aspectj-v2`                        | file `src/config/activate-it.tidalwave-aspectj-v2-profile`                           | AspectJ weaving                                                     |
| `it.tidalwave-aspectj-springaop-v1`              | file `src/config/activate-it.tidalwave-aspectj-springaop-v1-profile`                 | AspectJ weaving for Spring, compatible with Lombok                  |
| `it.tidalwave-aspectj-springaop-v2`              | file `src/config/activate-it.tidalwave-aspectj-springaop-v2-profile`                 | Same, newer variant                                                 |
| `javafx-run`                                     | `-P`                                                                                 | Run a JavaFX application on the module path                         |
| `it.tidalwave-javafx-appbundle-embedded-v2-*`    | file `src/config/activate-it.tidalwave-javafx-appbundle-embedded-v2-profile` (+ JDK) | macOS app bundles and DMGs with an embedded JRE                     |
| `it.tidalwave-monocle-profile-v1`                | `-P`                                                                                 | Headless JavaFX tests                                               |
| `it.tidalwave-deb-base-v1`                       | file `src/config/activate-it.tidalwave-deb-v1-profile`                               | Base configuration for `.deb` packages                              |
| `it.tidalwave-deb-service-v1`                    | file `src/config/activate-it.tidalwave-deb-service-v1-profile`                       | `.deb` for a systemd service                                        |
| `it.tidalwave-deb-application-v1`                | file `src/config/activate-it.tidalwave-deb-application-v1-profile`                   | `.deb` for a command-line application                               |
| `it.tidalwave.deb-sign`                          | property `gpg.keyname` defined                                                       | Sign `.deb` packages                                                |
| `it.tidalwave-spring-boot-single-v1`             | file `src/config/activate-it.tidalwave-spring-boot-single-v1`                        | Spring Boot executable JAR                                          |
| `it.tidalwave-spring-boot-native-v1`             | file `src/config/activate-it.tidalwave-spring-boot-native-v1`                        | GraalVM native executable                                           |
| `it.tidalwave-spring-boot-native-disabler`       | `$JAVA_HOME/bin/native-image` missing                                                | Skip native compilation on non-GraalVM JDKs                         |
| `it.tidalwave.use-local-artifact-target-repo-v1` | `-P`                                                                                 | Deploy to the local staging repository                              |
| `it.tidalwave.publish-artifacts-v1`              | `-P`                                                                                 | Copy staged artifacts to a remote repository (legacy)               |
| `it.tidalwave-update-releases-v1`                | `-P`                                                                                 | Update dependency version properties to the latest releases         |
| `it.tidalwave-update-snapshots-v1`               | `-P`                                                                                 | Update dependency version properties to the latest snapshots        |
| `it.tidalwave-update-headers`                    | `-P`                                                                                 | Update license headers                                              |
| `it.tidalwave-check-headers`                     | `-P`                                                                                 | Check license headers                                               |
| `it.tidalwave-generate-readme-md-v1`             | `-P`                                                                                 | Generate `README.md` from a template                                |
| `it.tidalwave-display-plugin-updates`            | `-P`                                                                                 | Report plugins that can be updated                                  |
| `javac-jdk9plus`                                 | JDK 9+                                                                               | Pass `--release` to the compiler                                    |
| `umldoclet-jdk9plus`                             | JDK 9+                                                                               | Use the UML Doclet for Javadoc                                      |
| `javac-jdk21plus`                                | JDK 21+                                                                              | Enable dynamic agent loading in tests                               |
| `it.tidalwave-m2e-lifecycle-mapping`             | property `m2e.version` (Eclipse)                                                     | m2e lifecycle mappings                                              |

The `-v1`, `-v2` suffixes are part of the profile ids: when the behaviour of a profile changes incompatibly, a new
version is added so that existing projects keep working.


### Continuous Integration, metrics and site

**`it.tidalwave-ci-v1`** is meant for CI builds: it excludes tests in the `no-ci` group and lets the build continue
when tests fail, so the CI server can report them.

**`it.tidalwave-metrics-v2`** runs a full QA pass. Its default goal (`tft.profile.metrics.defaultGoal`) is:

```
clean verify com.github.spotbugs:spotbugs-maven-plugin:check pmd:aggregate-cpd pmd:aggregate-pmd checkstyle:checkstyle dependency:analyze
sonar:sonar dependency-check:aggregate
```

Each step comes from its own property, which can be overridden or set empty to drop that step:
`tft.profile.metrics.spotbugsGoal`, `tft.profile.metrics.pmdGoal`, `tft.profile.metrics.checkstyleGoal`,
`tft.profile.metrics.dependencyGoal`, `tft.profile.metrics.sonarGoal`, `tft.profile.metrics.dependencyCheckGoal`
(`tft.profile.metrics.findbugsGoal` is empty by default). Sonar needs the usual `sonar.*` settings for your server.
The profile also:

* attaches the JaCoCo agent to Surefire and produces the coverage report at `test`;
* writes test reports to `target/surefire-reports/jacoco`;
* ignores test failures.

**`it.tidalwave-metrics-pre-site-v1`** empties the Checkstyle, SpotBugs, PMD and Dependency-Check goals of the metrics
profile. Activate it together with `it.tidalwave-metrics-v2` when the site is generated afterwards, to avoid running
those tools twice.

**`it.tidalwave-site-v1`** generates the site, with default goal `site site:stage`. It must be run after a build
(plain or with metrics), because some reports (Surefire, JaCoCo) are collected from the previous run while others are
created again. The site contains:

* project information reports (CI, dependencies, dependency convergence and management, distribution, issue management,
  licenses, mailing lists, modules, plugins, SCM, summary, team);
* aggregated Surefire report from both `surefire-reports/plain` and `surefire-reports/jacoco` (`tft.report.test.aggregate`,
  disable with `tft.maven-surefire-report-plugin.skip`);
* JaCoCo coverage of the module (for the root of a multi-module project, see `it.tidalwave-jacoco-aggregate-v1` below);
* SpotBugs, aggregated PMD and CPD, Checkstyle, OWASP Dependency-Check and RevAPI reports;
* JXR cross-referenced sources;
* Javadoc for main and test code, with an `aggregate` report set that runs only at the root of a multi-module project.

The site can be disabled with `tft.site.skip=true`. **`it.tidalwave-site-docs-maven-skin-v1`** adds the
[Docs Maven Skin](https://github.com/Bernardo-MG/docs-maven-skin) (`tft.site.skin.version`, default `2.1.0`) as a
dependency of the Site Plugin; this project activates it and configures it in `src/site/site.xml`.

**`it.tidalwave-jacoco-aggregate-v1`** is activated automatically in the root of a multi-module project, recognised by
the presence of a `modules` directory. When the site is generated it creates a JaCoCo report covering all the modules,
listed as *JaCoCo Aggregate* in *Project Reports* and written to `jacoco-aggregate` in the site output directory (HTML
and `jacoco.xml`). As for the other coverage reports, data come from a previous build with `it.tidalwave-metrics-v2`:
the profile reads every `target/jacoco.exec` under the modules directory, together with the classes and sources of the
modules placed directly in it (`modules/*`) or in its `examples` subdirectory (`modules/examples/*`); modules at other
depths are not included in the report. The `report-aggregate` goal of JaCoCo isn't used for this, because it only
aggregates the dependencies of a project and the root doesn't depend on its modules: the goal only creates the menu
entry, and its empty output is replaced by an Ant task of `org.jacoco.ant`. The directory scanned for the data can be
changed with `tft.modules.directory`, but the activation always looks for `modules`, because Maven doesn't interpolate
properties in file activation.

**`generate-revision-id`** writes a description of the build (`DESCRIPTION: <GIT_BRANCH> <version> <buildNumber>`) to
the file named by `tft.descriptionStamp` (default `.description`) and prints it together with the Java version.
**`show-revision-id`** prints the description previously stored in that file. Both run at `validate` and were designed
for Hudson/Jenkins builds.


### The release cycle

The release is **transactional**: the release is prepared and all artifacts are built, signed and staged on the local
disk, and nothing is pushed or uploaded. A second, separate step publishes everything or throws it away.

**1. Prepare and perform.** The `release` profile must be active when running the Release Plugin, because it defines
the `altDeploymentRepository` and `dryRun` properties passed to the forked builds:

```shell
mvn -Prelease release:prepare release:perform
```

The Release Plugin is configured as follows:

| Setting                        | Property                       | Default                                                                                       |
|--------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------|
| Local checkout                 | `tft.release.localCheckout`    | `true` (perform checks out from the local repository)                                         |
| Push changes                   | `tft.release.pushChanges`      | `false` (commits and tag stay local)                                                          |
| Preparation goals              | `tft.release.preparationGoals` | `clean install verify`                                                                        |
| Perform goals                  | `tft.release.goals`            | `clean install source:jar-no-fork source:test-jar-no-fork javadoc:jar assembly:single deploy` |
| Completion goals               | `tft.release.completionGoals`  | empty                                                                                         |
| Tag name                       | `tft.release.tagNameFormat`    | `@{project.version}`                                                                          |
| Profiles for the forked builds | `release.profiles`             | `release-sign-artifacts`                                                                      |

During the release:

* tests are skipped and the build number is dropped from the final name;
* `target/target-maven-repo` is cleaned at `initialize` and used as the staging repository (`staging.mvn.repo.url`);
* `release-sign-artifacts` signs all artifacts with GPG at `verify`, using `gpg.keyname` and `gpg.passphrase`. Set
  `tft.release.sign-artifacts.skip=true` to skip signing;
* the Sonatype Central Publishing Plugin (registered as a build extension) takes over the `deploy` phase and creates the
  bundle `target/central-publishing/central-bundle.zip`. Since `tft.central-publishing.skipUpload` is `true`, the
  bundle is **not** uploaded. The deployment is named `${project.name} ${project.version}` and is published
  automatically once uploaded (`tft.central-publishing.autoPublish`).

**2a. Commit to Maven Central.** If the release is good, run from the project root:

```shell
mvn -N -Prelease-commit-central
```

It reads the credentials of the `central` server from `settings.xml`, builds a bearer token from them (with a Groovy
script run by GMavenPlus), uploads `target/checkout/target/central-publishing/central-bundle.zip` to the Central Portal
API with `curl` and then pushes commits and tags (`git push origin master --tags`, or `hg push`).

**2b. Commit to another repository.** `mvn -N -Prelease-commit` copies the staged repository to the repository in
`<distributionManagement>` with the Wagon Maven Plugin, then pushes the SCM changes in the same way.

**2c. Cancel.** `mvn -N -Prelease-cancel` deletes the SCM metadata in `target/.git` (or `target/.hg`). It was designed for
Jenkins workspaces: **don't run it in your local work area**.

The SCM is detected by `scm-detector-git` and `scm-detector-hg`, which are active when `.git` or `.hg` exists.

Related profiles:

* `it.tidalwave.use-local-artifact-target-repo-v1` sets `altDeploymentRepository` to the local staging repository, so a
  normal `deploy` also stages instead of uploading;
* `it.tidalwave.publish-artifacts-v1` copies the staging repository to a remote repository with
  `mvn -N -Pit.tidalwave.publish-artifacts-v1` (`wagon.source`, `wagon.target`, `wagon.targetId`). Its default target is
  the legacy OSSRH snapshot repository. On macOS run it with `-Djava.io.tmpdir=/tmp`.


### Assemblies and extra artifacts

The Assembly Plugin uses GNU long file mode for tarballs and, by default, runs only at the execution root.

* **`it.tidalwave-assembly-bin-descriptor`** (file `src/assemble/bin.xml`) and
  **`it.tidalwave-assembly-project-descriptor`** (file `src/assemble/project.xml`) register those descriptors and tell the
  Build Helper Plugin to attach `${finalName}-bin.tar.gz` and `${finalName}-project.tar.gz` with classifiers `bin` and
  `project`. They don't bind anything to the lifecycle: the assembly runs when `assembly:single` is invoked (as in the
  release goals). In this project, `bin` contains `README*`, `LICENSE*` and `pom.xml`, and `project` contains the whole
  source tree except `target`, `.git` and `.description`.
* **`it.tidalwave-deploy-extra-artifacts-v2`** builds and deploys those assemblies with the default goal
  `initialize assembly:single build-helper:attach-artifact verify deploy:deploy`, staging into
  `../../target-maven-repo`.
* **`it.tidalwave-assembly-tar.gz-v1`** (file `src/main/assembly/tar.gz.xml`) and **`it.tidalwave-assembly-zip-v1`**
  (file `src/main/assembly/zip.xml`) run their descriptor at `package` in every module.
* **`it.tidalwave-distribution-v1`** prepares the files for an installer at `generate-resources`: filtered
  `src/main/resources` go to `target/dist`, runtime dependencies to `target/dist/lib`, and OpenJFX modules
  (`org.openjfx`) to `target/dist/mods`.
* **`it.tidalwave-manifest-with-main-v1`** (file `src/config/it.tidalwave-manifest-with-main-v1`, note: no `activate-`
  prefix) adds `Main-Class: ${tft.mainClass}` to the JAR manifest.


### Web applications

* **`tomcat`** configures the Tomcat 7 Maven Plugin with server id `tft.webapp.deployServerId`, URL
  `tft.webapp.deployServerUrl` and context path `tft.webapp.contextPath` (default `/`).
* **`it.tidalwave-jetty-v2`** configures the Jetty Maven Plugin (Jetty 11): host `tft.jetty.host` (`0.0.0.0`), port
  `tft.jetty.port` (`8080`), stop port `tft.jetty.stopPort` (`9966`), stop key `tft.jetty.stopKey`, scan interval
  `tft.jetty.scanIntervalSeconds` (`4`), work directory `tft.jetty.tmpDir` (`target/jetty-work`) and context path
  `tft.webapp.contextPath`. The WAR to run is a NorthernWind web application taken from the local Maven repository,
  identified by the properties `nw.webApp` and `nw.version`, which you must define. The antrun execution that creates
  the work directory (`create-jetty-tmpdir`) is not bound to a phase.
* **`it.tidalwave-webresources-v1`** packages `src/main/webapp` (filtered) into the `webapp` folder of the classpath,
  next to `src/main/resources`, for applications with an embedded web server.
* **`vaadin-compile-widgetset`** (`-P`) compiles custom Vaadin widgets with the GWT Maven Plugin into
  `src/main/webapp/VAADIN/widgetsets`, using GWT `tft.gwt.version` (default `2.1.1`).


### AspectJ and Spring AOP

All three profiles weave already compiled classes (they don't compile sources with `ajc`), use `spring-aspects` as an
aspect library, ignore Xlint warnings and use `tft.javac.source` / `tft.javac.target` as compliance levels. Set
`tft.aspectj.ajc.verbose=true` to see weaving information.

* **`it.tidalwave-aspectj-v2`** weaves `target/classes` and `target/test-classes` in place at `process-classes` and
  `process-test-classes`.
* **`it.tidalwave-aspectj-springaop-v1`** and **`it.tidalwave-aspectj-springaop-v2`** make AspectJ compatible with other
  tools that manipulate the bytecode, such as annotation processors (e.g. Lombok): `javac` writes to
  `target/unwoven-classes` and `target/unwoven-test-classes` (`tft.project.build.unwoven-classes.directory`,
  `tft.project.build.unwoven-test-classes.directory`), and the woven classes are written to the usual output
  directories. They also add `spring-aspects` and `aspectjweaver` as dependencies. The differences: v1 excludes
  `aspectjweaver` from `spring-aspects`, adds it with compile scope and includes a `com.sun:tools` workaround for old
  plugin versions on JDK 9+; v2 adds `aspectjweaver` with runtime scope and drops the workaround.

You must define `tft.aspectjrt.version` and manage the versions of `org.springframework:spring-aspects` and
`org.aspectj:aspectjweaver` (e.g. in `<dependencyManagement>`).


### JavaFX and macOS application bundles

**`javafx-run`** runs a modular JavaFX application with `mvn -Pjavafx-run` (`exec:exec@javafx-run`): it starts `java`
with the project classpath as module path, `--add-modules ALL-MODULE-PATH` and `-m ${tft.mainClass}`, so
`tft.mainClass` must be in the form `module/fully.qualified.MainClass`.

The **embedded app bundle** profiles create macOS application bundles for both Intel and Apple Silicon, each with its own
embedded Java runtime. They are all activated by the file
`src/config/activate-it.tidalwave-javafx-appbundle-embedded-v2-profile`, and the JDK running the build selects the
embedded runtime:

| Profile                                                 | JDK      | Role                                                                                                                                                                                                                                    |
|---------------------------------------------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `it.tidalwave-javafx-appbundle-embedded-v2-prepare`     | any      | At `generate-resources`, copies runtime dependencies to `target/app/lib` and unpacks the embedded JREs to `target/jre/intel` and `target/jre/arm`.                                                                                      |
| `it.tidalwave-javafx-appbundle-embedded-v2-jdk11`       | 11       | Selects the embedded JRE 11 (`tft.embedded-jre11.*`).                                                                                                                                                                                   |
| `it.tidalwave-javafx-appbundle-embedded-v2-jdk17`       | 17       | Selects the embedded JRE 17 (`tft.embedded-jre17.*`).                                                                                                                                                                                   |
| `it.tidalwave-javafx-appbundle-embedded-v2-jdk21`       | 21       | Selects the embedded JRE 21 (`tft.embedded-jre21.*`).                                                                                                                                                                                   |
| `it.tidalwave-javafx-appbundle-embedded-v2-jdk11-17-21` | 11 to 21 | At `package`, runs `jpackage` (`tft.tool.jpackage`, default `$JAVA_HOME/bin/jpackage`) twice to create `target/intel/<name>.app` and `target/arm/<name>.app`.                                                                           |
| `it.tidalwave-javafx-appbundle-embedded-v2-create-dmg`  | any      | At `pre-integration-test`, creates `<name>-intel.dmg` and `<name>-arm.dmg` (with `hdiutil` on macOS, or `tft.genisoimage` on Linux), gzips them and attaches them with classifier `executable` and types `intel.dmg.gz` / `arm.dmg.gz`. |

The embedded JREs are Maven artifacts with these coordinates:

| Coordinate                 | Property                                    | Default                            |
|----------------------------|---------------------------------------------|------------------------------------|
| Group id                   | `tft.embedded-jre.groupId`                  | `it.tidalwave.jre`                 |
| Artifact id                | `tft.embedded-jre.artifactId`               | `adoptium-jre`                     |
| Type                       | `tft.embedded-jre.type`                     | `tgz`                              |
| Classifier (Intel)         | `tft.embedded-jre.classifier.macos.x64`     | `macosx-x64-hotspot`               |
| Classifier (Apple Silicon) | `tft.embedded-jre.classifier.macos.aarch64` | `macosx-aarch64-hotspot`           |
| Version                    | `tft.embedded-jre.version`                  | set by the JDK profile (see below) |

The version, the directory name used when unpacking, and the name of the top-level directory inside the archive come
from the profile matching the JDK running the build:

| JDK | `tft.embedded-jre.version`   | `tft.embedded-jre.name`   | `tft.embedded-jre.prefix`   |
|-----|------------------------------|---------------------------|-----------------------------|
| 11  | `tft.embedded-jre11.version` | `tft.embedded-jre11.name` | `tft.embedded-jre11.prefix` |
| 17  | `tft.embedded-jre17.version` | `tft.embedded-jre17.name` | `tft.embedded-jre17.prefix` |
| 21  | `tft.embedded-jre21.version` | `tft.embedded-jre21.name` | `tft.embedded-jre21.prefix` |

The bundles are built only with JDK 11, 17 or 21, since other versions have no embedded runtime defined.

You must provide:

* `tft.appbundle.name`: the application name; the icon is read from `src/main/app-resources/<name>.icns`;
* `tft.mainModule` and `tft.mainClass`: the main module and class passed to `jpackage --module`.

The bundle identifier is `${groupId}.${artifactId}.${version}`, vendor and copyright come from
`project.organization.name`, and the bundle version is derived from the project version by dropping uppercase words and
replacing dashes with dots (e.g. `1.0-ALPHA-3-SNAPSHOT` becomes `1.0.3`). Logging verbosity is controlled by
`tft.appbundle.build.logLevel` (default `info`).


### Debian packages

`.deb` packages are built with [jdeb](https://github.com/tcurdt/jdeb) at `package`. The base profile must be combined with
either the service or the application profile.

**`it.tidalwave-deb-base-v1`** (file `src/config/activate-it.tidalwave-deb-v1-profile`) downloads a tarball and installs
its contents under `tft.deb.binaryDirectory`, adding the symlink `${tft.deb.binaryDirectory}/${tft.deb.name}` →
`${tft.deb.name}-${project.version}`. It also turns on filtering for `src/main/resources`. The tarball is identified by:

| Property                           | Default              |
|------------------------------------|----------------------|
| `tft.deb.name`                     | none, required       |
| `tft.deb.sourceTarball.groupId`    | `${project.groupId}` |
| `tft.deb.sourceTarball.artifactId` | none, required       |
| `tft.deb.sourceTarball.version`    | `${project.version}` |
| `tft.deb.sourceTarball.classifier` | `bin`                |
| `tft.deb.binaryDirectory`          | `/usr/lib`           |
| `tft.deb.user` / `tft.deb.group`   | `root` / `root`      |

File permissions are preserved from the tarball.

**`it.tidalwave-deb-service-v1`** (file `src/config/activate-it.tidalwave-deb-service-v1-profile`) adds:

* the systemd unit `src/main/resources/<name>.service` (filtered), installed to `/etc/systemd/system`;
* the directories `/var/lib/<dir>`, `/var/log/<dir>` and `/var/cache/<dir>`, where `<dir>` is `tft.deb.directory`
  (default `tft.deb.name`);
* the contents of `src/main/resources/config`, installed to `/var/lib/<dir>/config`.

**`it.tidalwave-deb-application-v1`** (file `src/config/activate-it.tidalwave-deb-application-v1-profile`) adds the
command `/usr/bin/<name>`, a symlink to `${tft.deb.binaryDirectory}/<name>/bin/<name>`.

**`it.tidalwave.deb-sign`** is active whenever `gpg.keyname` is defined, and signs the package with `dpkg-sig` (role
`origin`) using `gpg.keyname` and `gpg.passphrase`.


### Spring Boot and GraalVM native images

* **`it.tidalwave-spring-boot-single-v1`** (file `src/config/activate-it.tidalwave-spring-boot-single-v1`) repackages
  the application at `package` into an executable JAR with classifier `tft.spring-boot.repackage-classifier` (default
  `exec`), with main class `tft.mainClass`. Disable with `tft.spring-boot.skip=true`.
* **`it.tidalwave-spring-boot-native-v1`** (file `src/config/activate-it.tidalwave-spring-boot-native-v1`) builds a
  GraalVM native executable at `package` (`native:compile-no-fork`) named `tft.native.name` (default
  `${groupId}-${artifactId}-${version}`), with main class `tft.mainClass` and build arguments `tft.native.buildArgs`
  (default `--no-fallback`). Disable with `tft.native.skip=true`.
* **`it.tidalwave-spring-boot-native-disabler`** sets `tft.native.skip=true` automatically when
  `$JAVA_HOME/bin/native-image` doesn't exist, so the same build works on a regular JDK.


### Maintenance utilities

| Command                                       | Effect                                                                                                                                                           |
|-----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `mvn -N -Pit.tidalwave-update-releases-v1`    | Updates the version properties of the dependencies to their latest releases (`versions:update-properties`). Expects the standard TheseFoolishThings layout.      |
| `mvn -N -Pit.tidalwave-update-snapshots-v1`   | Same, allowing snapshots.                                                                                                                                        |
| `mvn -Pit.tidalwave-update-headers`           | Unpacks `superpom-config`, filters the header template with the project properties and updates the license header of each file.                                  |
| `mvn -Pit.tidalwave-check-headers`            | Same preparation, then checks the headers without changing them.                                                                                                 |
| `mvn -N -Pit.tidalwave-generate-readme-md-v1` | Regenerates `README.md` by filtering the template in `superpom-config` (`it/tidalwave/readme/`) with the project properties. **It overwrites the current file.** |
| `mvn -Pit.tidalwave-display-plugin-updates`   | Lists the plugins for which newer versions are available.                                                                                                        |


### Automatically activated profiles

* **`javac-jdk9plus`** (JDK 9+): passes `--release ${tft.javac.release}` to the compiler.
* **`umldoclet-jdk9plus`** (JDK 9+): uses the UML Doclet for Javadoc (see [Javadoc](#javadoc)).
* **`javac-jdk21plus`** (JDK 21+): adds `-XX:+EnableDynamicAgentLoading` to the test JVM.
* **`it.tidalwave-m2e-lifecycle-mapping`** (Eclipse m2e): maps the AspectJ (`compile`, `test-compile`), JAXB2 (`xjc`),
  XML (`transform`) and Dependency (`unpack`) goals to run on incremental builds, and ignores the Enforcer.
* **`scm-detector-git`**, **`scm-detector-hg`**, **`it.tidalwave-jacoco-aggregate-v1`**,
  **`it.tidalwave-spring-boot-native-disabler`** and **`it.tidalwave.deb-sign`** are described above.


### Properties reference

Defaults are shown as defined in the POM; an empty value means the property is defined but empty.

**General**

| Property                              | Default                                               | Description                                                                  |
|---------------------------------------|-------------------------------------------------------|------------------------------------------------------------------------------|
| `tft.defaultGoal`                     | `clean install`                                       | Goals run by a plain `mvn`                                                   |
| `tft.maven.requiredVersion`           | `3.9.0`                                               | Minimum Maven version                                                        |
| `tft.maven.requiredJdk`               | `[11,)`                                               | Accepted JDK range                                                           |
| `tft.superpom.config.version`         | `1.11`                                                | Version of `superpom-config` (Checkstyle, PMD, license and README resources) |
| `tft.mainClass`                       | placeholder                                           | Main class for `exec:java`, manifest, Spring Boot, native image, JavaFX      |
| `tft.jar.classpathInManifest`         | `false`                                               | Add `Class-Path` to the manifest                                             |
| `tft.disableDeploy`                   | `false`                                               | Skip `deploy`                                                                |
| `tft.disableJavaDocs`                 | `false`                                               | Skip Javadoc                                                                 |
| `tft.site.skip`                       | `false`                                               | Skip the site                                                                |
| `tft.buildNumber.getRevisionOnlyOnce` | `false`                                               | Query the SCM revision only once in a reactor                                |
| `tft.descriptionStamp`                | `.description`                                        | File used by `generate-revision-id` / `show-revision-id`                     |
| `tft.staging-repo.url`                | `https://central.sonatype.com`                        | Central Portal base URL                                                      |
| `tft.staging-repo.id`                 | `central`                                             | Server id for the Central Portal                                             |
| `staging.mvn.repo.url`                | `file://${project.build.directory}/target-maven-repo` | Local staging repository                                                     |

**Compiler**

| Property                    | Default      | Description                                                               |
|-----------------------------|--------------|---------------------------------------------------------------------------|
| `tft.javac.source`          | `11`         | `-source` level; also the PMD target JDK and the AspectJ compliance level |
| `tft.javac.target`          | `11`         | `-target` level                                                           |
| `tft.javac.release`         | `11`         | `--release` level, passed only on JDK 9+ (`javac-jdk9plus`)               |
| `tft.javac.xlint`           | `-Xlint:all` | Lint option, added after `-Xlint:-processing`                             |
| `tft.javac.debug`           | `true`       | Generate debug information                                                |
| `tft.javac.showDeprecation` | `true`       | Show deprecation warnings                                                 |
| `tft.javac.showWarnings`    | `true`       | Show compiler warnings                                                    |
| `tft.javac.fork`            | `true`       | Run the compiler in a separate process                                    |
| `tft.modulePathInTests`     | `true`       | Use the module path when compiling and running tests                      |

**Tests**

| Property                                | Default                                                        | Description                                                                          |
|-----------------------------------------|----------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `tft.test.skip`                         | `${skipTests}` (`false`)                                       | Skip tests                                                                           |
| `tft.test.maxMemory`                    | `768M`                                                         | Maximum heap of the test JVM (`-Xmx`)                                                |
| `tft.test.vmOptions`                    | empty                                                          | Extra options for the test JVM                                                       |
| `tft.test.allowAttachSelf`              | `-Djdk.attach.allowAttachSelf=true`                            | Lets agents attach to the test JVM; on JDK 21+ also `-XX:+EnableDynamicAgentLoading` |
| `tft.test.excludedGroups`               | empty                                                          | Test groups to exclude (`no-ci` in `it.tidalwave-ci-v1`)                             |
| `tft.test.failureIgnore`                | `${testFailureIgnore}` (`false`)                               | Don't fail the build when tests fail                                                 |
| `tft.test.reportsDirectory`             | `${project.build.directory}/surefire-reports/plain`            | Surefire report directory (`.../jacoco` in the metrics profile)                      |
| `tft.test.jacoco.argLine`               | empty                                                          | Set by JaCoCo in `it.tidalwave-metrics-v2`; don't set it yourself                    |
| `tft.report.test.aggregate`             | `true`                                                         | Aggregate the Surefire reports in the site                                           |
| `tft.maven-surefire-report-plugin.skip` | `false`                                                        | Skip the Surefire report in the site                                                 |
| `tft.openjfx-monocle.version`           | `11.0.2`                                                       | Monocle version (`it.tidalwave-monocle-profile-v1`)                                  |
| `tft.testfx-monocle.vmOptions`          | `-Dtestfx.robot=glass -Dtestfx.headless=true -Dprism.order=sw` | Headless options (`it.tidalwave-monocle-profile-v1`)                                 |

**Quality tools**

| Property                            | Default                           | Description                                            |
|-------------------------------------|-----------------------------------|--------------------------------------------------------|
| `tft.spotbugs.skip`                 | `${tft.findbugs.skip}` (`false`)  | Skip SpotBugs                                          |
| `tft.spotbugs.effort`               | `Max`                             | Analysis effort                                        |
| `tft.spotbugs.threshold`            | `Low`                             | Minimum confidence of reported bugs                    |
| `tft.spotbugs.failOnError`          | `false`                           | Fail the build when bugs are found                     |
| `tft.spotbugs.relaxed`              | `false`                           | Relaxed reporting mode                                 |
| `tft.spotbugs.maxHeap`              | `512`                             | Heap of the SpotBugs JVM, in MB                        |
| `tft.spotbugs.maxAllowedViolations` | `0`                               | Violations tolerated before failing                    |
| `tft.findbugs.skip`                 | `false`                           | Legacy switch; also the default of `tft.spotbugs.skip` |
| `tft.checkstyle.skip`               | `false`                           | Skip Checkstyle                                        |
| `tft.pmd.skip`                      | `false`                           | Skip PMD and CPD                                       |
| `tft.pmd.ruleset`                   | `/it/tidalwave/pmd/standard.xml`  | PMD ruleset resource                                   |
| `tft.pmd.ruleset.groupId`           | `it.tidalwave.thesefoolishthings` | Group id of the artifact containing the ruleset        |
| `tft.pmd.ruleset.artifactId`        | `superpom-config`                 | Artifact id of the artifact containing the ruleset     |
| `tft.pmd.ruleset.version`           | `${tft.superpom.config.version}`  | Version of the artifact containing the ruleset         |
| `tft.pmd.minimumTokens`             | `100`                             | CPD duplication threshold                              |
| `tft.pmd.minimumPriority`           | `5`                               | Lowest PMD rule priority reported                      |
| `tft.pmd.skipEmptyReport`           | `false`                           | Skip the PMD report when there are no violations       |
| `tft.owasp.skip`                    | `false`                           | Skip OWASP Dependency-Check                            |
| `tft.revapi.skip`                   | `true`                            | Skip the RevAPI compatibility check                    |

**Metrics profile goals** (`it.tidalwave-metrics-v2`; set a property empty to drop that step)

| Property                                  | Default                                           |
|-------------------------------------------|---------------------------------------------------|
| `tft.profile.metrics.spotbugsGoal`        | `com.github.spotbugs:spotbugs-maven-plugin:check` |
| `tft.profile.metrics.findbugsGoal`        | empty                                             |
| `tft.profile.metrics.pmdGoal`             | `pmd:aggregate-cpd pmd:aggregate-pmd`             |
| `tft.profile.metrics.checkstyleGoal`      | `checkstyle:checkstyle`                           |
| `tft.profile.metrics.dependencyGoal`      | `dependency:analyze`                              |
| `tft.profile.metrics.sonarGoal`           | `sonar:sonar`                                     |
| `tft.profile.metrics.dependencyCheckGoal` | `dependency-check:aggregate`                      |
| `tft.profile.metrics.defaultGoal`         | `clean verify` followed by all the goals above    |

**Javadoc**

| Property                             | Default                                 | Description                            |
|--------------------------------------|-----------------------------------------|----------------------------------------|
| `tft.javadoc.excludePackageNames`    | `*.impl:*.impl.*`                       | Packages excluded from Javadoc         |
| `tft.javadoc.quiet`                  | `true`                                  | Quiet output                           |
| `tft.javadoc.legacyMode`             | `false`                                 | Ignore modules and use the classpath   |
| `tft.javadoc.detectJavaApiLink`      | `false`                                 | Link to the JDK API documentation      |
| `tft.javadoc.doclint`                | empty                                   | Doclint option (e.g. `-Xdoclint:none`) |
| `tft.javadoc.doclet`                 | `nl.talsmasoftware.umldoclet.UMLDoclet` | Doclet class (JDK 9+)                  |
| `tft.javadoc.doclet.groupId`         | `nl.talsmasoftware`                     | Group id of the doclet artifact        |
| `tft.javadoc.doclet.artifactId`      | `umldoclet`                             | Artifact id of the doclet artifact     |
| `tft.javadoc.doclet.version`         | `2.2.2`                                 | Version of the doclet artifact         |
| `tft.javadoc.doclet.additionalParam` | empty                                   | Extra options for the doclet           |

**License headers**

| Property               | Default                                                  | Description                              |
|------------------------|----------------------------------------------------------|------------------------------------------|
| `tft.license.name`     | `tidalwave_asf2`                                         | License, resolved from `superpom-config` |
| `tft.license.template` | `/it/tidalwave/license/tidalwave-header-description.ftl` | Header template                          |

**Release and publishing**

| Property                                | Default                                                                                       | Description                                                           |
|-----------------------------------------|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| `tft.release.localCheckout`             | `true`                                                                                        | `release:perform` checks out from the local repository                |
| `tft.release.pushChanges`               | `false`                                                                                       | Push commits and tag during `release:prepare`                         |
| `tft.release.preparationGoals`          | `clean install verify`                                                                        | Goals run by `release:prepare`                                        |
| `tft.release.goals`                     | `clean install source:jar-no-fork source:test-jar-no-fork javadoc:jar assembly:single deploy` | Goals run by `release:perform`                                        |
| `tft.release.completionGoals`           | empty                                                                                         | Goals run after the release commit                                    |
| `tft.release.tagNameFormat`             | `@{project.version}`                                                                          | Name of the release tag                                               |
| `release.profiles`                      | `release-sign-artifacts`                                                                      | Profiles activated in the forked release builds                       |
| `tft.release.sign-artifacts.skip`       | `false`                                                                                       | Skip GPG signing                                                      |
| `tft.central-publishing.autoPublish`    | `true`                                                                                        | Publish automatically once the bundle is uploaded to Central          |
| `tft.central-publishing.skipPublishing` | `false`                                                                                       | Exclude the module from the bundle                                    |
| `tft.central-publishing.skipUpload`     | `true`                                                                                        | Only create the bundle; upload it later with `release-commit-central` |

**macOS application bundles**

| Property                                                                                     | Default                                                                           | Description |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|-------------|
| `tft.appbundle.name`                                                                         | empty, required                                                                   | Application name; icon read from `src/main/app-resources/<name>.icns` |
| `tft.mainModule`                                                                             | not defined, required                                                             | Main module passed to `jpackage` |
| `tft.appbundle.build.logLevel`                                                               | `info`                                                                            | Log level of the bundle and DMG steps |
| `tft.tool.jpackage`                                                                          | `${env.JAVA_HOME}/bin/jpackage`                                                   | Path to `jpackage` |
| `tft.genisoimage`                                                                            | `genisoimage`                                                                     | Tool used to create DMGs on Linux |
| `tft.embedded-jre.*`, `tft.embedded-jre11.*`, `tft.embedded-jre17.*`, `tft.embedded-jre21.*` | see [JavaFX and macOS application bundles](#javafx-and-macos-application-bundles) | Embedded runtime |

**Debian packages**

| Property                           | Default              | Description                                                                 |
|------------------------------------|----------------------|-----------------------------------------------------------------------------|
| `tft.deb.name`                     | empty, required      | Package name                                                                |
| `tft.deb.sourceTarball.groupId`    | `${project.groupId}` | Group id of the source tarball                                              |
| `tft.deb.sourceTarball.artifactId` | empty, required      | Artifact id of the source tarball                                           |
| `tft.deb.sourceTarball.version`    | `${project.version}` | Version of the source tarball                                               |
| `tft.deb.sourceTarball.classifier` | `bin`                | Classifier of the source tarball                                            |
| `tft.deb.binaryDirectory`          | `/usr/lib`           | Installation directory                                                      |
| `tft.deb.user`                     | `root`               | Owner of the installed files                                                |
| `tft.deb.group`                    | `root`               | Group of the installed files                                                |
| `tft.deb.directory`                | `${tft.deb.name}`    | Name of the `/var/lib`, `/var/log`, `/var/cache` directories (service only) |

The defaults of the `sourceTarball` properties (except `artifactId`), `binaryDirectory`, `user` and `group` are set by
`it.tidalwave-deb-base-v1`; `tft.deb.directory` by `it.tidalwave-deb-service-v1`.

**Spring Boot and native images**

| Property                               | Default                                                       | Description                      |
|----------------------------------------|---------------------------------------------------------------|----------------------------------|
| `tft.spring-boot.skip`                 | `false`                                                       | Skip the Spring Boot repackaging |
| `tft.spring-boot.repackage-classifier` | `exec`                                                        | Classifier of the executable JAR |
| `tft.native.skip`                      | `false` (`true` without GraalVM)                              | Skip the native image            |
| `tft.native.name`                      | `${project.groupId}-${project.artifactId}-${project.version}` | Name of the native executable    |
| `tft.native.buildArgs`                 | `--no-fallback`                                               | Arguments for `native-image`     |

**Web applications**

| Property                        | Default                                    | Description                                      |
|---------------------------------|--------------------------------------------|--------------------------------------------------|
| `tft.webapp.deployServerId`     | empty                                      | Server id in `settings.xml` (`tomcat`)           |
| `tft.webapp.deployServerUrl`    | empty                                      | Manager URL of the server (`tomcat`)             |
| `tft.webapp.contextPath`        | `/`                                        | Context path (`tomcat`, `it.tidalwave-jetty-v2`) |
| `tft.jetty.host`                | `0.0.0.0`                                  | Address Jetty listens on                         |
| `tft.jetty.port`                | `8080`                                     | HTTP port                                        |
| `tft.jetty.stopPort`            | `9966`                                     | Port used to stop Jetty                          |
| `tft.jetty.stopKey`             | `${project.groupId}:${project.artifactId}` | Key used to stop Jetty                           |
| `tft.jetty.scanIntervalSeconds` | `4`                                        | Interval for scanning changes                    |
| `tft.jetty.tmpDir`              | `${project.build.directory}/jetty-work`    | Jetty work directory                             |
| `tft.gwt.version`               | `2.1.1`                                    | GWT version (`vaadin-compile-widgetset`)         |

**AspectJ**

| Property                                           | Default                                           | Description                                                      |
|----------------------------------------------------|---------------------------------------------------|------------------------------------------------------------------|
| `tft.aspectjrt.version`                            | not defined, required                             | Version of `aspectjweaver` and `aspectjtools` used by the plugin |
| `tft.aspectj.ajc.verbose`                          | `false`                                           | Show weaving information                                         |
| `tft.project.build.unwoven-classes.directory`      | `${project.build.directory}/unwoven-classes`      | `javac` output in the Spring AOP profiles                        |
| `tft.project.build.unwoven-test-classes.directory` | `${project.build.directory}/unwoven-test-classes` | `javac` test output in the Spring AOP profiles                   |

**Site**

| Property                | Default   | Description                                                                         |
|-------------------------|-----------|-------------------------------------------------------------------------------------|
| `tft.site.skin.version` | `2.1.0`   | Docs Maven Skin version (`it.tidalwave-site-docs-maven-skin-v1`)                    |
| `tft.modules.directory` | `modules` | Directory of the modules, relative to the root (`it.tidalwave-jacoco-aggregate-v1`) |


### Managed plugin versions

Every version can be overridden by setting the corresponding property.

| Plugin                                                       | Version property                                |
|--------------------------------------------------------------|-------------------------------------------------|
| `com.github.spotbugs:spotbugs-maven-plugin`                  | `tft.spotbugs-maven-plugin.version`             |
| `com.mycila:license-maven-plugin`                            | `tft.mycila-license-maven-plugin.version`       |
| `com.simpligility.maven.plugins:android-maven-plugin`        | `tft.android-maven-plugin.version`              |
| `com.spotify:docker-maven-plugin`                            | `tft.docker-maven-plugin.version`               |
| `com.spotify:dockerfile-maven-plugin`                        | `tft.dockerfile-maven-plugin.version`           |
| `com.sun.xml.ws:jaxws-maven-plugin`                          | `tft.jaxws-maven-plugin.version`                |
| `com.vaadin:vaadin-maven-plugin`                             | `tft.vaadin-maven-plugin.version`               |
| `org.apache.felix:maven-bundle-plugin`                       | `tft.maven-bundle-plugin.version`               |
| `org.apache.maven.plugins:maven-antrun-plugin`               | `tft.maven-antrun-plugin.version`               |
| `org.apache.maven.plugins:maven-assembly-plugin`             | `tft.maven-assembly-plugin.version`             |
| `org.apache.maven.plugins:maven-checkstyle-plugin`           | `tft.maven-checkstyle-plugin.version`           |
| `org.apache.maven.plugins:maven-clean-plugin`                | `tft.maven-clean-plugin.version`                |
| `org.apache.maven.plugins:maven-compiler-plugin`             | `tft.maven-compiler-plugin.version`             |
| `org.apache.maven.plugins:maven-dependency-plugin`           | `tft.maven-dependency-plugin.version`           |
| `org.apache.maven.plugins:maven-deploy-plugin`               | `tft.maven-deploy-plugin.version`               |
| `org.apache.maven.plugins:maven-doap-plugin`                 | `tft.maven-doap-plugin.version`                 |
| `org.apache.maven.plugins:maven-enforcer-plugin`             | `tft.maven-enforcer-plugin.version`             |
| `org.apache.maven.plugins:maven-gpg-plugin`                  | `tft.maven-gpg-plugin.version`                  |
| `org.apache.maven.plugins:maven-install-plugin`              | `tft.maven-install-plugin.version`              |
| `org.apache.maven.plugins:maven-jar-plugin`                  | `tft.maven-jar-plugin.version`                  |
| `org.apache.maven.plugins:maven-jarsigner-plugin`            | `tft.maven-jarsigner-plugin.version`            |
| `org.apache.maven.plugins:maven-javadoc-plugin`              | `tft.maven-javadoc-plugin.version`              |
| `org.apache.maven.plugins:maven-jxr-plugin`                  | `tft.maven-jxr-plugin.version`                  |
| `org.apache.maven.plugins:maven-pmd-plugin`                  | `tft.maven-pmd-plugin.version`                  |
| `org.apache.maven.plugins:maven-project-info-reports-plugin` | `tft.maven-project-info-reports-plugin.version` |
| `org.apache.maven.plugins:maven-release-plugin`              | `tft.maven-release-plugin.version`              |
| `org.apache.maven.plugins:maven-resources-plugin`            | `tft.maven-resources-plugin.version`            |
| `org.apache.maven.plugins:maven-scm-plugin`                  | `tft.maven-scm-plugin.version`                  |
| `org.apache.maven.plugins:maven-shade-plugin`                | `tft.maven-shade-plugin.version`                |
| `org.apache.maven.plugins:maven-site-plugin`                 | `tft.maven-site-plugin.version`                 |
| `org.apache.maven.plugins:maven-source-plugin`               | `tft.maven-source-plugin.version`               |
| `org.apache.maven.plugins:maven-surefire-plugin`             | `tft.maven-surefire-plugin.version`             |
| `org.apache.maven.plugins:maven-surefire-report-plugin`      | `tft.maven-surefire-report-plugin.version`      |
| `org.apache.maven.plugins:maven-war-plugin`                  | `tft.maven-war-plugin.version`                  |
| `org.apache.tomcat.maven:tomcat7-maven-plugin`               | `tft.tomcat-maven-plugin.version`               |
| `org.codehaus.gmavenplus:gmavenplus-plugin`                  | hard-coded (in `release-commit-central`)        |
| `org.codehaus.mojo:aspectj-maven-plugin`                     | `tft.aspectj-maven-plugin.version`              |
| `org.codehaus.mojo:build-helper-maven-plugin`                | `tft.build-helper-maven-plugin.version`         |
| `org.codehaus.mojo:buildnumber-maven-plugin`                 | `tft.buildnumber-maven-plugin.version`          |
| `org.codehaus.mojo:exec-maven-plugin`                        | `tft.exec-maven-plugin.version`                 |
| `org.codehaus.mojo:findbugs-maven-plugin`                    | `tft.findbugs-maven-plugin.version`             |
| `org.codehaus.mojo:gwt-maven-plugin`                         | `tft.gwt-maven-plugin.version`                  |
| `org.codehaus.mojo:jaxb2-maven-plugin`                       | `tft.jaxb2-maven-plugin.version`                |
| `org.codehaus.mojo:license-maven-plugin`                     | `tft.license-maven-plugin.version`              |
| `org.codehaus.mojo:versions-maven-plugin`                    | `tft.versions-maven-plugin.version`             |
| `org.codehaus.mojo:wagon-maven-plugin`                       | `tft.wagon-maven-plugin.version`                |
| `org.codehaus.mojo:xml-maven-plugin`                         | `tft.xml-maven-plugin.version`                  |
| `org.eclipse.jetty:jetty-maven-plugin`                       | `tft.jetty-maven-plugin.version`                |
| `org.fusesource.mvnplugins:maven-graph-plugin`               | `tft.maven-graph-plugin.version`                |
| `org.graalvm.buildtools:native-maven-plugin`                 | `tft.native-maven-plugin.version`               |
| `org.jacoco:jacoco-maven-plugin`                             | `tft.jacoco-maven-plugin.version`               |
| `org.owasp:dependency-check-maven`                           | `tft.dependency-check-maven-plugin.version`     |
| `org.revapi:revapi-maven-plugin`                             | `tft.revapi-maven-plugin.version`               |
| `org.sonarsource.scanner.maven:sonar-maven-plugin`           | `tft.sonar-maven-plugin.version`                |
| `org.sonatype.central:central-publishing-maven-plugin`       | `tft.central-publishing-maven-plugin.version`   |
| `org.springframework.boot:spring-boot-maven-plugin`          | `tft.spring-boot-maven-plugin.version`          |
| `org.vafer:jdeb`                                             | `tft.maven-jdeb-plugin.version`                 |

Other versions:

| Component                                                              | Version property                     |
|------------------------------------------------------------------------|--------------------------------------|
| `org.revapi:revapi-java` (RevAPI analyzer)                             | `tft.revapi-java.version`            |
| `org.apache.maven.wagon:wagon-webdav-jackrabbit` (build extension)     | `tft.webdav-jackrabbit.version`      |
| `org.eclipse.m2e:lifecycle-mapping`                                    | `tft.m2e-lifecycle-mapping.version`  |
| `maven-archetype-plugin`                                               | `tft.maven-archetype-plugin.version` |
| Archetype packaging (defaults to `tft.maven-archetype-plugin.version`) | `tft.archetype-packaging.version`    |
| UML Doclet                                                             | `tft.javadoc.doclet.version`         |


### Properties you must provide

Some features reference properties that the SuperPOM doesn't define (or defines empty). Set them when you use the
corresponding feature:

| Property                                                  | Needed by                                                                                               |
|-----------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `tft.mainClass`                                           | `exec:java`, `it.tidalwave-manifest-with-main-v1`, `javafx-run`, app bundles, Spring Boot, native image |
| `tft.mainModule`                                          | app bundles                                                                                             |
| `tft.appbundle.name`                                      | app bundles and DMGs                                                                                    |
| `tft.aspectjrt.version`                                   | all AspectJ profiles                                                                                    |
| `tft.deb.name`, `tft.deb.sourceTarball.artifactId`        | Debian profiles                                                                                         |
| `tft.webapp.deployServerId`, `tft.webapp.deployServerUrl` | `tomcat`                                                                                                |
| `nw.webApp`, `nw.version`                                 | `it.tidalwave-jetty-v2`                                                                                 |
| `gpg.keyname`, `gpg.passphrase`                           | artifact and `.deb` signing                                                                             |
| `tft.pmd.includeXmlInSite`                                | PMD report in `it.tidalwave-site-v1`                                                                    |
| `central` server credentials in `settings.xml`            | `release-commit-central` and Central publishing                                                         |


### Deprecated and unused properties

The following old properties still work but are kept only for backward compatibility; use the `tft.*` equivalents:

| Deprecated                      | Replacement                                           |
|---------------------------------|-------------------------------------------------------|
| `skipTests`                     | `tft.test.skip` (`-DskipTests` still works as usual)  |
| `testFailureIgnore`             | `tft.test.failureIgnore`                              |
| `disableJavaDocs`               | `tft.disableJavaDocs`                                 |
| `disableDeploy`                 | `tft.disableDeploy`                                   |
| `stableAPI`, `stableAPIVersion` | `tft.stableAPI`, `tft.stableAPIVersion`               |
| `tft.javafx.vmargs`             | `tft.appbundle.args`                                  |

These properties are defined but not currently used by any plugin configuration:

| Property                      | Default                                         |
|-------------------------------|-------------------------------------------------|
| `tft.stableAPI`               | `${stableAPI}` (`false`)                        |
| `tft.stableAPIVersion`        | `${stableAPIVersion}` (`(,${project.version})`) |
| `tft.appbundle.args`          | `${tft.javafx.vmargs}` (empty)                  |
| `tft.appbundle.vmoptions`     | empty                                           |
| `tft.mainPackage`             | placeholder                                     |
| `tft.profile.metrics.jxrGoal` | `jxr:aggregate`                                 |
| `tft.project.description`     | `TheseFoolishThings Super POM`                  |
| `bundle.namespace`            | `${project.groupId}.${project.artifactId}`      |
| `netbeans.checkstyle.format`  | `true` (read by the NetBeans IDE, not by Maven) |
