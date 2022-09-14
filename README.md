# Best Practices when using Gradle

## General

### Don't use internal APIs

Gradle and many plugins (such as AGP) consider internal APIs fair game for making breaking changes
in even minor releases. Therefore, using such an API is inherently fragile and will lead to major,
completely avoidable, headaches. If you really need some functionality, it is often better to copy
relevant bits over to your codebase.

### Avoid making any ordering assumptions of any kind

[Lazy configuration](https://docs.gradle.org/current/userguide/lazy_configuration.html), callbacks,
and provider chains are the name of the game.

## Laziness

### Don't do expensive computations in the configuration phase

It slows down the build. Such computations should be encapsulated in a task action.

### Avoid the [`create`](https://docs.gradle.org/current/javadoc/org/gradle/api/NamedDomainObjectContainer.html#create-java.lang.String-org.gradle.api.Action-) method on Gradle's container types

Use [register](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskContainer.html#register-java.lang.String-java.lang.Class-org.gradle.api.Action-) instead.

### Avoid the [`all`](https://docs.gradle.org/current/javadoc/org/gradle/api/DomainObjectCollection.html#all-org.gradle.api.Action-) callback on Gradle's container types

These cause object to be intialized eagerly. Use configureEach instead.

### Don't assume your plugin is applied after another

Apply order can be arbitary, instead use pluginManager.withPlugin() to reach
when plugins are added.

### Don't call `get()` on a `Provider` outside a task action

The whole point of using a provider is to evaluate it as late as possible. Calling `get()`—evaluating
it—will lead to painful ordering issues if done too early. Instead, use map or flatMap.

## Cacheability

### Make all tasks and transforms cacheable (with some exceptions)

Sadly, Gradle default is to not to cache any tasks or transforms. Use `@CacheableTask` and
`@CacheableTransform`. The exceptions are:
* copy/package(jar/zip)/unpackage(extract) since generally it is faster to rerun this task locally
than downloading/unpacking it from the cache.
* input is non-stable (time, git sha, etc) as you will have little to none cache hits.

### Don't access a `Project` instance inside a task action

It breaks the configuration cache, and will eventually be deprecated. Instead, specify the exact
inputs you were using `Project` instance for as an explicit task input.

### Don't access another project's Project instance

This is called cross-project configuration and is extremely fragile. It creates implicit, nearly
un-modelable dependencies between projects and can only lead to grief. Instead, share artifacts
across projects by declaring dependencies.

It also breaks the experimental project isolation feature, but that won't be truly relevant for a while.

### No overlapping output files and directories between tasks

Two tasks having the same output file or directly will likely result in constant build cache invalidation.
Instead set unique outputs for every task, especially when creating per variant/flavor tasks.