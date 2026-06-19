# Migration Notes

Curated dispositions for every site the scanner flagged that was **not** mechanically
rewritten. All `[CONFIRMED]` removed-accessor (Cat-A) call sites and the genuinely-real
changed-return-type (Cat-B), Groovy operator (Cat-C), and collection-op (Cat-E) sites have
already been edited directly in the tree (see the task commit). What remains below are
name-collision false positives and a set of unconfirmed Cat-A setter calls whose receiver the
scanner cannot import-confirm; the genuinely-removed ones among the latter surface as concrete
`./gradlew help` configuration/compile errors and are resolved in task 07.

Each file in the index at the bottom carries one or more cluster tags. The clusters and the
site-specific reason for each are:

## `[FP-NAME]` — getter-name collisions on non-migrated receivers (Cat-B)

The scanner matches a changed getter *name* against every migrated property that shares it, then
marks the hit `[CONFIRMED]` whenever the file merely *imports* one such owning type. The actual
receiver at these sites is a different, non-migrated accessor that still returns a plain value, so
the read compiles unchanged. The recurring offenders, with their real owning type:

- `getName()` — `Project`/`Task`/`Named`/`Configuration`/`SourceSet`/`TestDescriptor.getName()`, all `String` (not `JacocoTaskExtension`/publication `name`).
- `getPath()` — `Project`/`Task.getPath()` (`String`) and `java.io.File.getPath()`; not a migrated property.
- `getConfigurations()` — `Project.getConfigurations()` → `ConfigurationContainer`; not `AbstractDependencyReportTask.configurations`.
- `getParameters()` — `ValueSource`/`BuildService`/`TransformAction.getParameters()`; not `GroovyCompileOptions.parameters`.
- `getVersion()` — `Project.getVersion()` (`Object`) and `ModuleVersionIdentifier`/`Dependency.getVersion()`; not `MavenPublication`/`StandardJavadocDocletOptions.version`.
- `getOutput()` — `SourceSet.getOutput()` → `SourceSetOutput`; not `JacocoTaskExtension.output`.
- `getType()` / `getId()` / `getModule()` / `getRevision()` / `getDisplayName()` / `getTarget()` / `getPort()` / `getUrl()` — component/identifier/dependency accessors on non-`org.gradle` or non-migrated types.
- `getRootDir()` — `Project.getRootDir()` (`File`); not `VersionControlRepository.rootDir`.
- `getSourceSets()` — container accessor, not a lazy property.

## `[FP-FILECOLL]` — `file_collection` getters read through the old supertype (Cat-B)

`getClasspath()` / `getFile()` / similar now return `ConfigurableFileCollection`, a **subtype** of the
old `FileCollection`. Existing reads (`.getFiles()`, `.getAsPath()`, `!= null`, assignment to a
`FileCollection` local, `.stream()`) bind to the supertype API and compile without change. Only the
removed *setters* needed migration, and those were handled as Cat-A. Also covered here:
`getDestinationDirectory()` (already a `DirectoryProperty` in the baseline) and
`getJvmArgumentProviders()` (already a `ListProperty`) — used with `.add(...)`/`.file(...)`, the
correct lazy forms.

## `[FP-MULTILINE]` — already-migrated sites flagged on the getter line

A few sites I rewrote put the `.set(...)` / `.get()` on the line *after* the getter, so the scanner
still flags the bare getter line: `GenerateMavenPom.getDestination()` (now `.set(buildDirectory.file(...))`),
`StandardJavadocDocletOptions.getLinksOffline()` (now `.get().stream()` / `.set(...)`), and
`ConfigurableFileCollection` reads I converted to `.setFrom(...)`/`.from(...)`. These are correct.

## `[FP-CATC]` — Groovy operator mutations on non-Gradle receivers (Cat-C)

- `excludes << '...'` under `tasks.named("licenseHeaders")` is the custom
  `org.elasticsearch.gradle.internal.conventions.precommit.LicenseHeadersTask.getExcludes()`, which
  returns a plain `List<String>` (not `JacocoTaskExtension.excludes`). `List.leftShift` is unaffected
  by the migration — left as-is.
- `arguments += [...]` in `x-pack/plugin/esql/build.gradle` mutates a **local** Groovy `def arguments = [...]`
  list (declared a few lines above and passed to `args ... as List<String>`), not `AntlrTask.arguments`.

The real Cat-C — `options.compilerArgs << '...'` (`CompileOptions.compilerArgs`, now `ListProperty`) — was
rewritten to `.add(...)` in every `build.gradle` that had it.

## `[FP-CATD]` — local variable declarations (Cat-D)

`prop = value` heuristics flagged Java local declarations whose name collides with a property:
`FileCollection classpath = task.getClasspath();` (subtype-compatible, see `[FP-FILECOLL]`) and
`List<String> args = new ArrayList<>();` (a fresh local list). These are not property assignments.

## `[DEFER07]` — unconfirmed removed-setter calls in build logic / DSL (Cat-A)

Removed `set*(...)` calls whose receiver the scanner could not import-confirm — `setStandardOutput`,
`setErrorOutput`, `setIgnoreExitValue`, `setCommandLine`, `setExecutable`, `setArgs`, `setClasspath`,
`setTestClassesDirs`, `setIncludes`, `setUrl`, `setName` on exec/test/copy specs in `build.gradle`
scripts and convention `.gradle`/Java logic. Where the receiver really is the migrated Gradle type
these are genuine and produce a `MissingMethodException`/`cannot find symbol` at `./gradlew help`;
they are fixed in task 07 against that exact error output (the mechanical lazy-setter rewrite). They
are listed here rather than guessed at because confirming each receiver requires the compiler.

## `[DEFER07-FIXTURE]` — setter names inside test fixtures (Cat-A)

The same `set*` names occur inside `src/test`/`src/integTest` `*.groovy`/`*FuncTest`/`*Spec` files,
almost always as **build-script text embedded in a GString fixture** (e.g. `buildFile << """ ... setClasspath(...) ... """`)
or against the func-test harness's own types — not live Gradle API on this build's classpath. Verified
by location; revisited only if task 07 proves one is live build logic.

## File index

Format: `` `path` [clusters] — topGetters ``. Every file the scanner still flags appears here.
- `build-tools/src/main/java/org/elasticsearch/gradle/jarhell/JarHellPlugin.java` [DEFER07,FP-NAME] — classpath*1,configurations*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/SplitPackagesAuditPrecommitPlugin.java` [DEFER07,FP-NAME] — classpath*1,configurations*1,path*1
- `build-tools-internal/src/main/groovy/elasticsearch.fips.gradle` [DEFER07,FP-NAME] — classpath*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/docker/DockerValueSource.java` [DEFER07,FP-NAME] — commandLine*1,standardOutput*1,errorOutput*1,ignoreExitValue*1
- `build-tools/src/main/java/org/elasticsearch/gradle/plugin/StablePluginBuildPlugin.java` [DEFER07,FP-NAME] — configurations*2,classpath*1,output*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ThirdPartyAuditPrecommitPlugin.java` [DEFER07,FP-NAME] — configurations*5,outputDir*1,path*1
- `distribution/docker/build.gradle` [DEFER07,FP-NAME] — destinationDir*2,compression*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/foreign/ForeignLibraryPlugin.java` [DEFER07,FP-NAME] — destinationDirectory*3,source*1,annotationProcessorPath*1,output*1
- `build-tools/src/main/java/org/elasticsearch/gradle/plugin/GenerateNamedComponentsTask.java` [DEFER07,FP-NAME] — errorOutput*1,standardOutput*1,classpath*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalDistributionBwcSetupPlugin.java` [DEFER07,FP-NAME] — name*11,parameters*7,configurations*3,type*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/LoggerUsagePrecommitPlugin.java` [DEFER07,FP-NAME] — name*2,classpath*1,configurations*1,sourceSets*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/JdkDownloadPlugin.java` [DEFER07,FP-NAME] — name*2,configurations*1,version*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalDistributionArchiveSetupPlugin.java` [DEFER07,FP-NAME] — name*2,configurations*2,compression*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/PublishPlugin.java` [DEFER07,FP-NAME] — name*2,description*2,url*1,version*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalBwcGitPlugin.java` [DEFER07,FP-NAME] — name*2,type*1,output*1,path*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/testfixtures/TestFixturesDeployPlugin.java` [DEFER07,FP-NAME] — name*3,tags*1
- `build-tools/src/main/java/org/elasticsearch/gradle/DistributionDownloadPlugin.java` [DEFER07,FP-NAME] — name*4,configurations*4,type*4,version*4
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ElasticsearchTestBasePlugin.java` [DEFER07,FP-NAME] — name*9,configurations*5,showExceptions*1,showCauses*1
- `build-conventions/build.gradle` [DEFER07,FP-NAME] — output*1,classpath*1,file*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/JavaModulePrecommitPlugin.java` [DEFER07,FP-NAME] — output*2,classpath*1,configurations*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ForbiddenApisPrecommitPlugin.java` [DEFER07,FP-NAME] — outputDir*1,classpath*1,output*1,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/CheckForbiddenApisTask.java` [DEFER07,FP-NAME] — parameters*11,classpath*2,includes*1,excludes*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ThirdPartyAuditTask.java` [DEFER07,FP-NAME] — path*2,executable*2,standardOutput*2,ignoreExitValue*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/testfixtures/TestFixturesPlugin.java` [DEFER07,FP-NAME] — path*2,projectName*1,name*1
- `modules/repository-azure/build.gradle` [DEFER07,FP-NAME] — testClassesDirs*1,classpath*1,output*1
- `modules/repository-gcs/build.gradle` [DEFER07,FP-NAME] — testClassesDirs*1,classpath*1,output*1
- `modules/transport-netty4/build.gradle` [DEFER07,FP-NAME] — testClassesDirs*1,classpath*1,output*1
- `build-tools-internal/src/main/groovy/elasticsearch.base.gradle` [DEFER07,FP-NAME] — version*1,path*1
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/ElasticsearchNode.java` [DEFER07,FP-NAME] — version*21,name*6,file*3,distributionType*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/info/GlobalBuildInfoPlugin.java` [DEFER07,FP-NAME] — version*3,javaVersion*2,displayName*2,name*1
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/ElasticsearchCluster.java` [DEFER07,FP-NAME] — version*4,name*2,configurations*2
- `build-tools/src/testFixtures/groovy/org/elasticsearch/gradle/fixtures/JdkToolchainTestFixture.groovy` [DEFER07-FIXTURE,FP-NAME] — executable*1,arguments*1
- `build-tools/src/testFixtures/groovy/org/elasticsearch/gradle/fixtures/DistributionDownloadFixture.groovy` [DEFER07-FIXTURE,FP-NAME] — url*1,arguments*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/ClusterFeaturesMetadataPlugin.java` [DEFER07-FIXTURE] — classpath*1
- `build-tools/src/testFixtures/java/org/elasticsearch/gradle/internal/test/BuildConfigurationAwareGradleRunner.java` [DEFER07-FIXTURE] — debug*1
- `build-tools/src/testFixtures/java/org/elasticsearch/gradle/internal/test/InternalAwareGradleRunner.java` [DEFER07-FIXTURE] — debug*1
- `build-tools/src/testFixtures/java/org/elasticsearch/gradle/internal/test/NormalizeOutputGradleRunner.java` [DEFER07-FIXTURE] — debug*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/rest/compat/compat/AbstractYamlRestCompatTestPlugin.java` [DEFER07-FIXTURE] — destinationDir*1,enabled*1
- `build-tools-internal/src/test/java/org/elasticsearch/gradle/internal/precommit/FilePermissionsTaskTests.java` [DEFER07-FIXTURE] — executable*1
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/test/rest/LegacyYamlRestTestPluginFuncTest.groovy` [DEFER07-FIXTURE] — executable*3
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/MutedTestPlugin.java` [DEFER07-FIXTURE] — failOnNoMatchingTests*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/rest/CopyRestApiTask.java` [DEFER07-FIXTURE] — includes*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/rest/CopyRestTestsTask.java` [DEFER07-FIXTURE] — includes*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/LegacyRestTestBasePlugin.java` [DEFER07-FIXTURE] — maxParallelForks*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/rest/RestTestBasePlugin.java` [DEFER07-FIXTURE] — maxParallelForks*1,version*1,type*1
- `test/framework/src/main/java/org/elasticsearch/test/ESTestCase.java` [DEFER07-FIXTURE] — name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/TestWithSslPlugin.java` [DEFER07-FIXTURE] — outputDir*1
- `build-tools-internal/src/test/java/org/elasticsearch/gradle/internal/ConcatFilesTaskTests.java` [DEFER07-FIXTURE] — target*2
- `build-tools/src/main/java/org/elasticsearch/gradle/test/JavaRestTestPlugin.java` [DEFER07-FIXTURE] — testClassesDirs*1,classpath*1
- `build-tools/src/main/java/org/elasticsearch/gradle/test/YamlRestTestPlugin.java` [DEFER07-FIXTURE] — testClassesDirs*1,classpath*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/test/DistroTestPlugin.java` [DEFER07-FIXTURE] — type*1,version*1
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/InternalDistributionBwcSetupPluginFuncTest.groovy` [DEFER07-FIXTURE] — url*1
- `build-tools-internal/src/test/groovy/org/elasticsearch/gradle/internal/JdkSpec.groovy` [DEFER07-FIXTURE] — version*1
- `build-tools-internal/src/test/java/org/elasticsearch/gradle/internal/JdkDownloadPluginTests.java` [DEFER07-FIXTURE] — version*1
- `build-tools-internal/src/test/java/org/elasticsearch/gradle/AbstractDistributionDownloadPluginTests.java` [DEFER07-FIXTURE] — version*1,type*1
- `build-tools/src/test/groovy/org/elasticsearch/gradle/DistributionDownloadPluginSpec.groovy` [DEFER07-FIXTURE] — version*1,type*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/release/GitWrapper.java` [DEFER07] — commandLine*1,standardOutput*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/transport/TransportVersionResourcesService.java` [DEFER07] — commandLine*2,standardOutput*2,errorOutput*2,ignoreExitValue*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/dra/DraResolvePlugin.java` [DEFER07] — name*1,url*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/RepositoriesSetupPlugin.java` [DEFER07] — name*2
- `build-tools/gradle-runner/src/main/java/org/elasticsearch/gradle/runner/GradleRunner.java` [DEFER07] — standardOutput*1
- `x-pack/plugin/esql/build.gradle` [FP-CATC] — arguments*2
- `build-tools-internal/build.gradle` [FP-CATC] — excludes*1
- `x-pack/plugin/prometheus/build.gradle` [FP-CATC] — excludes*1
- `x-pack/plugin/otel-data/build.gradle` [FP-CATC] — excludes*2
- `server/build.gradle` [FP-CATC] — excludes*3
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/precommit/LicenseHeadersPrecommitPluginFuncTest.groovy` [FP-NAME,FP-CATC] — name*3,path*1,excludes*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/docker/NativeImageBuildTask.java` [FP-NAME,FP-CATD] — classpath*2,mainClass*1,outputFile*1,parameters*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ElasticsearchJavaModulePathPlugin.java` [FP-NAME,FP-CATD] — id*4,classpath*4,configurations*1,path*1
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/MockApmServer.java` [FP-NAME] — address*2,port*2,path*1,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/LoggerUsageTask.java` [FP-NAME] — classpath*2,parameters*2,output*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/packer/CacheCacheableTestFixtures.java` [FP-NAME] — classpath*3,name*3,parameters*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ElasticsearchJavaBasePlugin.java` [FP-NAME] — compilerArgs*1,configurations*1
- `x-pack/plugin/core/template-resources/build.gradle` [FP-NAME] — configurations*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/packer/CacheTestFixtureResourcesPlugin.java` [FP-NAME] — configurations*1,sourceSets*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ElasticsearchJavaPlugin.java` [FP-NAME] — configurations*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/DependencyLicensesPrecommitPlugin.java` [FP-NAME] — configurations*2
- `build-tools/src/main/java/org/elasticsearch/gradle/dependencies/CompileOnlyResolvePlugin.java` [FP-NAME] — configurations*2,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/DependenciesInfoPlugin.java` [FP-NAME] — configurations*3
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ElasticsearchJavadocPlugin.java` [FP-NAME] — configurations*3,path*3,classpath*2,source*2
- `build-tools/src/main/java/org/elasticsearch/gradle/util/GradleUtils.java` [FP-NAME] — configurations*5,output*4,name*3,module*2
- `build-tools/src/main/java/org/elasticsearch/gradle/plugin/BasePluginBuildPlugin.java` [FP-NAME] — configurations*7,output*2,path*1,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalDistributionModuleCheckTaskProvider.java` [FP-NAME] — destinationDir*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ForbiddenPatternsTask.java` [FP-NAME] — excludes*1,name*1
- `build-tools/src/main/java/org/elasticsearch/gradle/LoggedExec.java` [FP-NAME] — executable*1,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/CheckstylePrecommitPlugin.java` [FP-NAME] — file*2,version*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/SymbolicLinkPreservingTar.java` [FP-NAME] — file*4
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/dependencies/rules/ExcludeOtherGroupsTransitiveRule.java` [FP-NAME] — id*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/util/DependenciesUtils.java` [FP-NAME] — id*2,displayName*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/info/DefaultBuildParameterExtension.java` [FP-NAME] — javaVersion*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/info/GitInfo.java` [FP-NAME] — name*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/PomValidationPrecommitPlugin.java` [FP-NAME] — name*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/PrecommitTask.java` [FP-NAME] — name*1
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/fixtures/AbstractGradleInternalPluginFuncTest.groovy` [FP-NAME] — name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalTestArtifactPlugin.java` [FP-NAME] — name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/Jdk.java` [FP-NAME] — name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/doc/SnippetParserException.java` [FP-NAME] — name*1
- `build-tools/src/main/java/org/elasticsearch/gradle/LazyPropertyMap.java` [FP-NAME] — name*1
- `build-tools/src/main/java/org/elasticsearch/gradle/plugin/PluginBuildPlugin.java` [FP-NAME] — name*1
- `build-tools/src/main/java/org/elasticsearch/gradle/transform/SymbolicLinkPreservingUntarTransform.java` [FP-NAME] — name*1
- `x-pack/qa/repository-old-versions/build.gradle` [FP-NAME] — name*1
- `build-tools/src/main/java/org/elasticsearch/gradle/jarhell/JarHellTask.java` [FP-NAME] — name*1,classpath*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/EmbeddedProviderExtension.java` [FP-NAME] — name*1,configurations*1,output*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/docker/DockerSupportPlugin.java` [FP-NAME] — name*1,rootDir*1
- `build-tools/src/main/java/org/elasticsearch/gradle/plugin/PluginPropertiesExtension.java` [FP-NAME] — name*1,version*1
- `build-tools/src/main/java/org/elasticsearch/gradle/plugin/GenerateTestBuildInfoTask.java` [FP-NAME] — name*11
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/TestingConventionsCheckTask.java` [FP-NAME] — name*15,parameters*3,classpath*2,testClassesDirs*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/BuildPlugin.java` [FP-NAME] — name*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ValidateJsonAgainstSchemaTask.java` [FP-NAME] — name*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/release/BundleChangelogsTask.java` [FP-NAME] — name*2
- `build-tools/src/main/java/org/elasticsearch/gradle/transform/UnzipTransform.java` [FP-NAME] — name*2
- `x-pack/qa/rolling-upgrade-multi-cluster/build.gradle` [FP-NAME] — name*2
- `build-tools/src/main/java/org/elasticsearch/gradle/transform/FilteringJarTransform.java` [FP-NAME] — name*2,parameters*2,excludes*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/doc/DocSnippetTask.java` [FP-NAME] — name*3
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/DependencyLicensesTask.java` [FP-NAME] — name*3
- `build-tools/src/main/java/org/elasticsearch/gradle/LazyPropertyList.java` [FP-NAME] — name*3
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/FormattingPrecommitPlugin.java` [FP-NAME] — name*3,configurations*1,path*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/dependencies/patches/hdfs/HdfsClassPatcher.java` [FP-NAME] — name*3,parameters*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/NoticeTask.java` [FP-NAME] — name*4
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/dependencies/patches/azurecore/AzureCoreClassPatcher.java` [FP-NAME] — name*4
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ValidateJsonNoKeywordsTask.java` [FP-NAME] — name*4,file*1
- `build-tools/src/main/java/org/elasticsearch/gradle/transform/UnpackTransform.java` [FP-NAME] — name*4,parameters*4,path*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalTestArtifactExtension.java` [FP-NAME] — name*5,configurations*4
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/esql/EsqlFunctionPlugin.java` [FP-NAME] — name*6,path*2,rootDir*2,file*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalDistributionArchiveCheckPlugin.java` [FP-NAME] — name*8,destinationDir*3,file*1
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/SymbolicLinkPreservingTarFuncTest.groovy` [FP-NAME] — name*9
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/BuildPluginFuncTest.groovy` [FP-NAME] — output*1
- `build-tools/src/testFixtures/groovy/org/elasticsearch/gradle/fixtures/AbstractGradleFuncTest.groovy` [FP-NAME] — output*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/transport/TransportVersionReferencesPlugin.java` [FP-NAME] — output*1,configurations*1,name*1,outputFile*1
- `build-tools/src/integTest/groovy/org/elasticsearch/gradle/LoggedExecFuncTest.groovy` [FP-NAME] — output*2
- `distribution/packages/build.gradle` [FP-NAME] — output*2
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/precommit/ForbiddenPatternsPrecommitPluginFuncTest.groovy` [FP-NAME] — output*3
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/precommit/ThirdPartyAuditPrecommitPluginFuncTest.groovy` [FP-NAME] — output*3,file*1
- `build-tools-internal/src/integTest/groovy/org/elasticsearch/gradle/internal/precommit/TestingConventionsPrecommitPluginFuncTest.groovy` [FP-NAME] — output*4
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/MrjarPlugin.java` [FP-NAME] — output*5,sourceSets*3,configurations*1,destinationDirectory*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/snyk/GenerateSnykDependencyGraph.java` [FP-NAME] — outputFile*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/VersionPropertiesBuildService.java` [FP-NAME] — parameters*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/info/GitInfoValueSource.java` [FP-NAME] — parameters*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/info/TestSeedValueSource.java` [FP-NAME] — parameters*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/util/ParamsUtils.java` [FP-NAME] — parameters*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/VersionPropertiesPlugin.java` [FP-NAME] — parameters*1,properties*1
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/TestClusterValueSource.java` [FP-NAME] — parameters*2
- `build-tools/src/main/java/org/elasticsearch/gradle/ReaperPlugin.java` [FP-NAME] — parameters*3
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/transport/TransportVersionResourcesPlugin.java` [FP-NAME] — parameters*3,configurations*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/docker/DockerBuildTask.java` [FP-NAME] — parameters*3,name*1,tags*1
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/TestClustersPlugin.java` [FP-NAME] — parameters*3,path*2,name*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/docker/DockerSupportService.java` [FP-NAME] — parameters*4
- `build-tools/src/main/java/org/elasticsearch/gradle/ReaperService.java` [FP-NAME] — parameters*4,file*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ElasticsearchBuildCompletePlugin.java` [FP-NAME] — parameters*4,gradleVersion*1,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/BwcSetupExtension.java` [FP-NAME] — parameters*4,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/SplitPackagesAuditTask.java` [FP-NAME] — parameters*5,name*4,path*1,classpath*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/DraSnapshotBuildIdValueSource.java` [FP-NAME] — parameters*8
- `build-tools-internal/src/main/groovy/elasticsearch.bc-upgrade-test.gradle` [FP-NAME] — path*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ProjectSubscribeBuildService.java` [FP-NAME] — path*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/JarHellPrecommitPlugin.java` [FP-NAME] — path*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/release/UpdateBranchesJsonTask.java` [FP-NAME] — path*1
- `qa/full-cluster-restart/build.gradle` [FP-NAME] — path*1
- `qa/mixed-cluster/build.gradle` [FP-NAME] — path*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/distribution/ElasticsearchDistributionExtension.java` [FP-NAME] — path*1,configurations*1,name*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/LicenseHeadersTask.java` [FP-NAME] — path*1,excludes*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/snyk/SnykDependencyMonitoringGradlePlugin.java` [FP-NAME] — path*1,name*1,version*1,gradleVersion*1
- `modules/reindex/build.gradle` [FP-NAME] — path*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/RestrictedBuildApiService.java` [FP-NAME] — path*2,parameters*1,name*1
- `build-tools-internal/src/main/groovy/elasticsearch.ide.gradle` [FP-NAME] — path*2,rootDir*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/JavaModulePrecommitTask.java` [FP-NAME] — path*3
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/TestClustersRegistry.java` [FP-NAME] — path*4,name*4,parameters*2
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/TestClustersAware.java` [FP-NAME] — path*4,parameters*3,name*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/BaseInternalPluginBuildPlugin.java` [FP-NAME] — path*5,configurations*2,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/JarApiComparisonTask.java` [FP-NAME] — path*5,name*1
- `build-tools/src/main/java/org/elasticsearch/gradle/testclusters/RunTask.java` [FP-NAME] — port*2,rootDir*1,properties*1,name*1
- `build.gradle` [FP-NAME] — propertiesFile*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/release/UpdateVersionsTask.java` [FP-NAME] — revision*2
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/transport/GenerateInitialTransportVersionTask.java` [FP-NAME] — revision*2
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/GitInfoPlugin.java` [FP-NAME] — revision*2,parameters*1,rootDir*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ForbiddenPatternsPrecommitPlugin.java` [FP-NAME] — rootDir*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/release/ReleaseToolsPlugin.java` [FP-NAME] — rootDir*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/release/PruneChangelogsTask.java` [FP-NAME] — rootDir*1,name*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/ValidateRestSpecPlugin.java` [FP-NAME] — rootDir*2
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/util/Util.java` [FP-NAME] — rootDir*3,sourceSets*1,name*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/LicenseHeadersPrecommitPlugin.java` [FP-NAME] — sourceSets*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/PrecommitPlugin.java` [FP-NAME] — sourceSets*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/PrecommitTaskPlugin.java` [FP-NAME] — sourceSets*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/StringTemplatePlugin.java` [FP-NAME] — sourceSets*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/doc/DocsTestPlugin.java` [FP-NAME] — sourceSets*1,output*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/precommit/TestingConventionsPrecommitPlugin.java` [FP-NAME] — sourceSets*1,output*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/info/ParallelDetector.java` [FP-NAME] — standardOutput*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/ConcatFilesTask.java` [FP-NAME] — target*3
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/release/ChangelogEntry.java` [FP-NAME] — title*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/InternalDistributionDownloadPlugin.java` [FP-NAME] — type*10,version*3,name*2
- `build-tools/src/main/java/org/elasticsearch/gradle/ElasticsearchDistribution.java` [FP-NAME] — type*6,name*3
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/precommit/PomValidationTask.java` [FP-NAME] — url*4,name*3,groupId*1,artifactId*1
- `build-conventions/src/main/java/org/elasticsearch/gradle/internal/conventions/LicensingPlugin.java` [FP-NAME] — version*2,revision*1
- `build-tools-internal/src/main/java/org/elasticsearch/gradle/internal/DependenciesInfoTask.java` [FP-NAME] — version*4,name*3,id*1
