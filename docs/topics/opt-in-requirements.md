[//]: # (title: Opt-in requirements)

The Kotlin standard library provides a mechanism for requiring and giving explicit consent to use certain API elements.
This mechanism allows library developers to inform users about specific conditions that require opt-in,
such as when an API is in an experimental state and is likely to change in the future. 

To protect users, the compiler warns about these conditions and requires them to opt in before the API can be used.

## Opt in to API

If a library author marks a declaration from their library's API as _[requiring opt-in](#require-opt-in-to-use-api)_,
you must give explicit consent before you can use it in your code.
There are several ways to opt in. We recommend choosing the approach that best suits your situation.

### Opt in locally

To opt in to a specific API element when you use it in your code, use the `@OptIn` annotation with a reference to the experimental
API marker. For example, suppose you want to use the `DateProvider` class, which requires an opt-in:

```kotlin
// Library code
@RequiresOptIn(message = "This API is experimental. It may be changed in the future without notice.")
@Retention(AnnotationRetention.BINARY)
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
annotation class MyDateTime

@MyDateTime                            
class DateProvider // A class requiring opt-in
```

In your code, before declaring a function that uses the `DateProvider` class, add the `@OptIn` annotation with
a reference to the `MyDateTime` annotation class:

```kotlin
// Client code
@OptIn(MyDateTime::class)
fun getDate(): Date { // Uses DateProvider
    val dateProvider: DateProvider
    // ...
}
```

It's important to note that with this approach, if the `getDate()` function is called elsewhere in your code or used by 
another developer, no opt-in is required:

```kotlin
// Client code
@OptIn(MyDateTime::class)
fun getDate(): Date { // Uses DateProvider
    val dateProvider: DateProvider
    // ...
}

fun displayDate() {
    println(getDate()) // OK: No opt-in is required
}
```

The opt-in requirement is not propagated in your code. This is risky because you and others could use experimental API 
without realizing it. In the worst case, you could end up building a supposedly stable API on top of an experimental one.
A safer approach is to propagate the opt-in requirements when you choose to opt in.

#### Propagate opt-in requirements

When you use API in your code that's intended for third-party use, such as in a library, you can propagate its opt-in requirement
to your API as well. To do this, mark your declaration with the same _[opt-in requirement annotation](#create-opt-in-requirement-annotations)_
used by the library.

For example, let's use the `DateProvider` class from before, which requires an opt-in:

```kotlin
// Library code
@RequiresOptIn(message = "This API is experimental. It may be changed in the future without notice.")
@Retention(AnnotationRetention.BINARY)
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
annotation class MyDateTime // Opt-in requirement annotation

@MyDateTime                            
class DateProvider // A class requiring opt-in
```

In your code, before declaring a function that uses the `DateProvider` class, add the `@MyDateTime` annotation:

```kotlin
// Client code
@MyDateTime
fun getDate(): Date {  
    val dateProvider: DateProvider // OK: the function requires opt-in as well
    // ...
}

fun displayDate() {
    println(getDate()) // Error: getDate() requires opt-in
}
```

As you can see in this example, the annotated function appears to be a part of the `@MyDateTime` API.
The opt-in propagates the opt-in requirement to users of the `getDate()` function.

Note, if you implicitly use an API element that has an opt-in requirement, you are still required to opt-in. For example,
if you use an API element that doesn't have an opt-in requirement annotation but its signature includes a type that requires
opt-in, using it triggers an error.

```kotlin
// Client code
fun getDate(dateProvider: DateProvider): Date { // Error: DateProvider requires opt-in
    // ...
}

fun displayDate() {
    println(getDate()) // Warning: the signature of getDate() contains DateProvider, which requires opt-in
}
```

Note that if `@OptIn` applies to the declaration whose signature contains a type declared as requiring opt-in,
the opt-in requirement still propagates:

```kotlin
// Client code
@OptIn(MyDateTime::class)
fun getDate(dateProvider: DateProvider): Date { // Has DateProvider as a part of a signature; propagates the opt-in requirement
    // ...
}

fun displayDate() {
    println(getDate()) // Error: getDate() requires opt-in
}
```

To use multiple APIs that require opt-in, mark the declaration with all their opt-in requirement annotations. For example:

```kotlin
@MyDateTime
@AnotherClass
```

### Opt in a file

To use an API that requires opt-in in all functions and classes in a file, add the file-level annotation `@file:OptIn`
to the top of the file before the package specification and imports.

 ```kotlin
 // Client code
 @file:OptIn(MyDateTime::class)
 ```

### Opt in a module

> The `-opt-in` compiler option is available since Kotlin 1.6.0. For earlier Kotlin versions, use `-Xopt-in`.
>
{type="note"}

If you don't want to annotate every usage of APIs that require opt-in, you can opt in to them for your whole module.
To opt in to using an API in a module, compile it with the argument `-opt-in`,
specifying the fully qualified name of the opt-in requirement annotation of the API you use: `-opt-in=org.mylibrary.OptInAnnotation`.
Compiling with this argument has the same effect as if every declaration in the module has the annotation`@OptIn(OptInAnnotation::class)`.

If you build your module with Gradle, you can add arguments like this:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompilationTask
// ...

tasks.named<KotlinCompilationTask<*>>("compileKotlin").configure {
    compilerOptions.freeCompilerArgs.add("-opt-in=org.mylibrary.OptInAnnotation")
}

```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
import org.jetbrains.kotlin.gradle.tasks.KotlinCompilationTask
// ...

tasks.named('compileKotlin', KotlinCompilationTask) {
    compilerOptions {
        freeCompilerArgs.add("-opt-in=org.mylibrary.OptInAnnotation")
    }
}
```

</tab>
</tabs>

If your Gradle module is a multiplatform module, use the `optIn` method:

<tabs group="build-script">
<tab title="Kotlin" group-key="kotlin">

```kotlin
sourceSets {
    all {
        languageSettings.optIn("org.mylibrary.OptInAnnotation")
    }
}
```

</tab>
<tab title="Groovy" group-key="groovy">

```groovy
sourceSets {
    all {
        languageSettings {
            optIn('org.mylibrary.OptInAnnotation')
        }
    }
}
```

</tab>
</tabs>

For Maven, use the following:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>kotlin-maven-plugin</artifactId>
            <version>${kotlin.version}</version>
            <executions>...</executions>
            <configuration>
                <args>
                    <arg>-opt-in=org.mylibrary.OptInAnnotation</arg>                    
                </args>
            </configuration>
        </plugin>
    </plugins>
</build>
```

To opt in to multiple APIs on the module level, add one of the described arguments for each opt-in requirement marker used in your module.

### Opt in to inherit from a class or interface

Sometimes, a library author provides an API but wants to require users to explicitly opt in before they can extend it. 
For example, the library API may be stable for use but not for inheritance, as it might be extended in the future with 
new abstract functions. Library authors can enforce this by marking [open](inheritance.md) or [abstract classes](classes.md#abstract-classes) and [non-functional interfaces](interfaces.md)
with the [`@SubclassOptInRequired`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-subclass-opt-in-required/) annotation.

To opt in to use such an API element and extend it in your code, use the `@SubclassOptInRequired` annotation
with a reference to the annotation class. For example, suppose you want to use the `CoreLibraryApi` interface, which 
requires an opt-in:

```kotlin
// Library code
@RequiresOptIn(
 level = RequiresOptIn.Level.WARNING,
 message = "Interfaces in this library are experimental"
)
annotation class UnstableApi()

@SubclassOptInRequired(UnstableApi::class)
interface CoreLibraryApi // An interface requiring opt-in to extend
```

In your code, before creating a new interface that inherits from the `CoreLibraryApi` interface, add the `@SubclassOptInRequired`
annotation with a reference to the `UnstableApi` annotation class:

```kotlin
// Client code
@SubclassOptInRequired(UnstableApi::class)
interface SomeImplementation : CoreLibraryApi
```

Note that when you use the `@SubclassOptInRequired` annotation on a class, the opt-in requirement is not propagated to 
any [inner or nested classes](nested-classes.md):

```kotlin
// Library code
@RequiresOptIn
annotation class ExperimentalFeature

@SubclassOptInRequired(ExperimentalFeature::class)
open class FileSystem {
    open class File
}

// Client code
class NetworkFileSystem : FileSystem() // Opt-in is required

// Nested class
class TextFile : FileSystem.File() // No opt-in required
```

Alternatively, you can also opt in by using the `@OptIn` annotation, or an annotation that references the opt-in annotation
class to propagate the requirement further in your code:

```kotlin
// Client code
// With @OptIn annotation
@OptInRequired(UnstableApi::class)
interface SomeImplementation : CoreLibraryApi

// With annotation referencing annotation class
// Propagates the opt-in requirement further
@UnstableApi
interface SomeImplementation : CoreLibraryApi
```

## Require opt-in to use API 

### Create opt-in requirement annotations

If you want to require explicit consent to use your module's API, create an annotation class to use as an _opt-in requirement annotation_.
This class must be annotated with [@RequiresOptIn](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-requires-opt-in/):

```kotlin
@RequiresOptIn
@Retention(AnnotationRetention.BINARY)
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
annotation class MyDateTime
```

Opt-in requirement annotations must meet several requirements. They must have:
* `BINARY` or `RUNTIME` [retention](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.annotation/-retention/).
* `EXPRESSION`, `FILE`, `TYPE`, or `TYPE_PARAMETER` as a [target](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.annotation/-target/).
* No parameters.

An opt-in requirement can have one of two severity [levels](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-requires-opt-in/-level/):
* `RequiresOptIn.Level.ERROR`. Opt-in is mandatory. Otherwise, the code that uses marked API won't compile. This is the default level.
* `RequiresOptIn.Level.WARNING`. Opt-in is not mandatory, but advisable. Without it, the compiler raises a warning.

To set the desired level, specify the `level` parameter of the `@RequiresOptIn` annotation.

Additionally, you can provide a `message` to inform API users about any special conditions of using the API. 
The compiler shows this message to users that try to use the API without opt-in.

```kotlin
@RequiresOptIn(level = RequiresOptIn.Level.WARNING, message = "This API is experimental. It can be incompatibly changed in the future.")
@Retention(AnnotationRetention.BINARY)
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
annotation class ExperimentalDateTime
```

If you publish multiple independent features that require opt-in, declare an annotation for each.
This makes using your API safer for your clients because they can use only the features that they explicitly accept.
This also means that you can remove the opt-in requirements from features independently, which makes your API easier
to maintain.

### Mark API elements

To require an opt-in to use an API element, annotate its declaration with an opt-in requirement annotation:

```kotlin
@MyDateTime
class DateProvider

@MyDateTime
fun getTime(): Time {}
```

Note that for some language elements, an opt-in requirement annotation is not applicable:

* You can't annotate a backing field or a getter of a property, just the property itself.
* You can't annotate a local variable or a value parameter.

## Require opt-in to extend API

There may be times when you want more granular control over which specific parts of your API can be used and
extended. For example, when you have some API that is stable to use but:

* **Unstable to implement** due to ongoing evolution, such as when you have a family of interfaces where you expect to add new abstract functions without default implementations.
* **Delicate or fragile to implement**, such as individual functions that need to behave in a coordinated manner.
* **Has a contract that may be weakened in the future** in a backwards-incompatible manner for external implementations, such as changing an input parameter `T` to a nullable version `T?` where the code didn't previously consider `null` values.

In such cases, you want to require users to opt in to your API before they can extend it further.
For example, by inheriting from the API or declaring implementations for abstract functions. Using the [`@SubclassOptInRequired`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-subclass-opt-in-required/)
annotation, you can enforce this behavior for [open](inheritance.md) or [abstract classes](classes.md#abstract-classes) and [non-functional interfaces](interfaces.md).

To add the opt-in requirement to an API element, use the [`@SubclassOptInRequired`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-subclass-opt-in-required/)
annotation with a reference to the annotation class:

```kotlin
@RequiresOptIn(
 level = RequiresOptIn.Level.WARNING,
 message = "Interfaces in this library are experimental"
)
annotation class UnstableApi()

@SubclassOptInRequired(UnstableApi::class)
interface CoreLibraryApi // An interface requiring opt-in to extend
```

Note that when you use the `@SubclassOptInRequired` annotation to require opt-in, the requirement is not propagated to
any [inner or nested classes](nested-classes.md).

For a real-world example of how to use the `@SubclassOptInRequired` annotation in your API, check out the
[`SharedFlow`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/)
interface in the `kotlinx.coroutines` library.

## Opt-in requirements for pre-stable APIs

If you use opt-in requirements for features that are not stable yet, carefully handle the API graduation to avoid 
breaking client code.

Once your pre-stable API graduates and is released in a stable state, remove any opt-in requirement annotations from 
your declarations. The clients can then use them without restriction. However, you should leave the annotation classes 
in modules so that the existing client code remains compatible.

To encourage API users to update their modules by removing any annotations from their code and recompiling, mark the annotations
as [`@Deprecated`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-deprecated/) and provide an explanation in the deprecation message.

```kotlin
@Deprecated("This opt-in requirement is not used anymore. Remove its usages from your code.")
@RequiresOptIn
annotation class ExperimentalDateTime
```