# Best Practices when using Gradle

## General

### Use the latest Gradle and plugin versions

Allows you to get all performance and feature improvements. You can set up [shadows jobs](https://slack.engineering/shadow-jobs/)
to help test against upcoming versions and catch any regressions in advance.

### Don't use internal APIs

Gradle and many plugins (such as AGP) consider internal APIs fair game for making breaking changes
in even minor releases. Therefore, using such an API is inherently fragile and will lead to major,
completely avoidable, headaches. If you really need some functionality, it is often better to copy
relevant bits over to your codebase.

### Avoid making any ordering assumptions of any kind

[Lazy configuration](https://docs.gradle.org/current/userguide/lazy_configuration.html), callbacks,
and provider chains are the name of the game.

### Avoid afterEvaluate

It introduces subtle ordering issues which can be very challenging to debug.

What you're looking for is probably a [`Provider`](https://docs.gradle.org/current/javadoc/org/gradle/api/provider/Provider.html) or [`Property`](https://docs.gradle.org/current/javadoc/org/gradle/api/provider/Property.html) (see also [lazy configuration](https://docs.gradle.org/current/userguide/lazy_configuration.html)).

### Create custom tasks

[Gradle documentation suggests to use generic tasks](https://docs.gradle.org/current/userguide/tutorial_using_tasks.html)
with [`doFirst`](https://docs.gradle.org/current/javadoc/org/gradle/api/Task.html#doFirst-org.gradle.api.Action-) or [`doLast`]( https://docs.gradle.org/current/javadoc/org/gradle/api/Task.html#doLast-org.gradle.api.Action-)
to do the work in - **DON'T!**. Even for simple tasks it is much better to create a custom task class as it lets you
specify inputs, outputs, and most importantly cacheability of the task (see [cacheability section](#make-all-tasks-and-transforms-cacheable-with-some-exceptions)).

```kotlin
abstract class MyTask: DefaultTask() {
    @get:InputFiles
    abstract val thingsToRead: ConfigurableFileCollection
    @get:OuputFile
    abstract val placeToWrite: RegularFileProperty
    @TaskAction
    fun doThings() = TODO()
}
```

Note, that making task and input/output properties abstract, Gradle will automatically initialize them for you without having to call
`project.objects` factory methods.

### Enable stricter Gradle plugin validation

Use [`ValidatePlugins`](https://docs.gradle.org/current/javadoc/org/gradle/plugin/devel/tasks/ValidatePlugins.html)
that is added by [`java-gradle-plugin`](https://docs.gradle.org/current/userguide/java_gradle_plugin.html)
and set
```kotlin
tasks.withType<ValidatePlugins>().configureEach {
    failOnWarning.set(true)
    enableStricterValidation.set(true)
}
```

## Dependencies

### Keep your dependencies clustered

Having `dependencies {}` block with the dependencies clustered by destination (main, test, androidTest, etc.) makes it easier for
others to see what's on the classpath and how it should be changed.

### Use appropriate configurations for dependencies

Prefer `implementation` over `api`. Add dependencies in places you use it instead of adding to all projects or configurations just in case.
Use [`dependency-analysis-android-gradle-plugin`](https://github.com/autonomousapps/dependency-analysis-android-gradle-plugin) to help
maintain clean dependency lists.

### Use [version catalogs](https://docs.gradle.org/current/userguide/platforms.html) for shared dependencies

Avoids having to change dozens of lines when you want to upgrade a version of a library. Reduced variation between versions used in
the projects can also help have more accurate test coverage.

## Laziness

### Don't do expensive computations in the configuration phase

It slows down the build. Such computations should be encapsulated in a task action.

### Avoid the [`create`](https://docs.gradle.org/current/javadoc/org/gradle/api/NamedDomainObjectContainer.html#create-java.lang.String-org.gradle.api.Action-) method on Gradle's container types

Use [`register`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskContainer.html#register-java.lang.String-java.lang.Class-org.gradle.api.Action-) instead.

### Avoid the [`all`](https://docs.gradle.org/current/javadoc/org/gradle/api/DomainObjectCollection.html#all-org.gradle.api.Action-) callback on Gradle's container types

These cause object to be initialized eagerly. Use `configureEach` instead.

### Don't assume your plugin is applied after another

Apply order can be arbitrary, instead use [`pluginManager.withPlugin()`](https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/PluginManager.html#withPlugin-java.lang.String-org.gradle.api.Action-) to reach when plugins are added.

### Don't call `get()` on a `Provider` outside a task action

The whole point of using a provider is to evaluate it as late as possible. Calling [`get()`](https://docs.gradle.org/current/javadoc/org/gradle/api/provider/Provider.html#get--) — evaluating
it — will lead to painful ordering issues if done too early. Instead, use map or flatMap.

## Cacheability

### Make all tasks and transforms cacheable (with some exceptions)

Sadly, Gradle default is to not cache any tasks or transforms. Use `@CacheableTask` and
`@CacheableTransform`. The exceptions are:
* copy/package(jar/zip)/unpackage(extract) since generally it is faster to rerun this task locally
than downloading/unpacking it from the cache.
* input is non-stable (time, git sha, etc.) as you will have little to none cache hits.

### Annotate your inputs and outputs

* File inputs should be annotated as [`@InputFile`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/InputFile.html) or [`@InputFiles`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/InputFiles.html) otherwise Gradle will not keep track on when these files change
and your task is out of date.
* Annotate properties that Gradle should not consider in task up to dateness with [`@Internal`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/Internal.html)

### Don't access a `Project` instance inside a task action

It breaks the configuration cache, and will eventually be deprecated. Instead, specify the exact
inputs you were using [`Project`](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html) instance for as an explicit task input.

### Don't access another project's `Project` instance

This is called cross-project configuration and is extremely fragile. It creates implicit, nearly
un-modelable dependencies between projects and can only lead to grief. Instead, share artifacts
across projects by declaring dependencies.

It also breaks the experimental project isolation feature, but that won't be truly relevant for a while.

### No overlapping output files and directories between tasks

Two tasks having the same output file or directly will likely result in constant build cache invalidation.
Instead set unique outputs for every task, especially when creating per variant/flavor tasks.

### Prefer `@PathSensitive(PathSensitivity.NONE)` for all file inputs

Sadly, Gradle default is to treat every file input as absolute path sensitive input. Instead, use
[`@PathSensitive(PathSensitivity.NONE)`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/PathSensitivity.html#NONE) as that let's Gradle know that you only care about the contents of the file
and not their location. Other reasonable normalizers are [`PathSensitivity.NAME_ONLY`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/PathSensitivity.html#NAME_ONLY), [`PathSensitivity.RELATIVE`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/PathSensitivity.html#RELATIVE),
or using [`@Classpath`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/Classpath.html).

### Make your tasks outputs deterministic

Consider sorting your inputs in a way that you have deterministic output for the same set of inputs.
For example, this can come up when doing directory traversal or receiving non-ordered collections.

### Do not use [`task.outputs.upToDateWhen`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskOutputs.html#upToDateWhen-org.gradle.api.specs.Spec-)

This API predates proper Gradle input/output handling, [use annotations instead](##annotate-your-inputs-and-outputs). The only reasoanble
usage of this API is `task.outputs.upToDateWhen { false }` for tasks that should always re-run, but ideally you have very few of those.

## Plugin public APIs (DSL)

### Use plugin extensions to define your public API

Instead of using Gradle, system, or Java properties [create Extension objects using extension container](https://docs.gradle.org/current/userguide/custom_plugins.html#sec:getting_input_from_the_build).
This will allow your users have a robust way of configuring your plugin.

### Don't use Kotlin lambdas in your public API

I know, it's tempting. They're right there. Use [`Action<T>`](https://docs.gradle.org/current/javadoc/org/gradle/api/Action.html)
instead. Gradle enhances the bytecode at runtime to provide a nicer DSL experience for users of your
plugin.

### Don't use lists in your custom extensions

Use [domain object containers](https://docs.gradle.org/current/javadoc/org/gradle/api/model/ObjectFactory.html#domainObjectContainer-java.lang.Class-)
instead. Once again, Gradle is able to provide enhanced DSL support this way.

## Testing

### Run your integration tests with `--warning-mode=fail`

Making all [warnings](https://docs.gradle.org/current/userguide/command_line_interface.html#sec:command_line_warnings) fail the builds under test by passing `--warning-mode=fail` to all your integration tests make sure your plugins don't use any deprecated Gradle features.
It prevents you and contributors from inadvertently introducing such usages.
Moreover, when you add a new Gradle version to your testing matrix you get direct feedback on what needs attention.

In the event you need to relax this for some test, do it gradually until you can resolve the problem.

## Credits

- [autonomousapps](https://github.com/autonomousapps) via [Tony's rules for Gradle Plugin Authors](https://dev.to/autonomousapps/tonys-rules-for-gradle-plugin-authors-28k3)
