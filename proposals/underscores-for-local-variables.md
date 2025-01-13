# Underscore syntax for unused local variables

* **Type**: Design proposal
* **Authors**: Leonid Startsev, Mikhail Zarechenskiy
* **Contributors**: Alejandro Serrano Mena, Marat Akhin, Nikita Bobko, Pavel Kunyavskiy
* **Status**: Experimental in Kotlin 2.2
* **Discussion**: [link to discussion thread or issue]

## Synopsis

Allow to use `_` (one underscore) as a local variable name to explicitly express that it is unused:

```kotlin
fun writeTo(file: File): Boolean {
    val result = runCatching { file.writeText("Hello World!") }
    return result.isSuccess
}

fun foo(file: File) {
    val _ = writeTo(file) // We are not interested whether write operation is successfull
}
```

Underscore can be used only in *local variable declaration*: `val _ = ...` and cannot be referenced or resolved otherwise.

## Motivation

1. Explicit expression of intent

When reading and reviewing code you are not very familiar with, it may be hard to say whether unused function return value is an intention or a subtle bug.
Explicit syntax for this would help quickly differentiate between the two.

2. Readiness for unused return value checker

In the scope of [TODO: INSERT URVC KEEP LINK], a smart checker for unused return values will be added to Kotlin compiler.
This syntax will make its adoption much easier, since there would be no need to `@Suppress` checker output in case value has to be explicitly ignored.

3. Uniformity in existing Kotlin features

Kotlin already supports underscores for unused things in a number of places — see [References and further extensions](#references-and-further-extensions) section of this document.
Adding underscore as a name for unused local variables is a logical expansion of this trend.

## Proposed syntax

It is allowed to declare a variable as `val _`. Such variable is called **unnamed**.
Unnamed variable declarations should adhere to the following rules:

* Only local variables (not class or top-level properties) can be declared unnamed.
* Only `val` keyword can be used, `var` is not allowed.
* You can have multiple unnamed variables in the single scope.
* Unnamed variables cannot be delegated.
* Referencing unnamed variable is not possible. Because of that, it is required to have an initializer.
* Unnamed variables can have their type specified explicitly. If type is not specified, it is inferred using the same rules as for regular variables.

### Discarded alternatives

Besides using `val _ = ...` syntax, a shorter `_ = ...` statement was considered as an alternative.
However, we've decided not to proceed with it for the following reasons:

1. `val` syntax is better aligned with already existing Kotlin features.

Kotlin already has positional-based destructuring with an option to use underscore to omit certain components: `val (first, _) = getSomePair()`. Since positional-based destructuring declares new variables, it uses `val` syntax.

On the other hand, Kotlin has 'underscore for unused lambda parameter' feature. Using underscore without `val` here may imply some connection with the lambda parameter, while in reality, there is no such connection:

```kotlin
listOf(1, 2, 3).mapIndexed { idx, _ ->
  _ = unusedCall(idx) // Is _ a new unnamed variable or the same lambda parameter?
}
```

2. `val` syntax implies creation of a new unnamed variable, rather than referencing existing one.

Normally, without `val/var` keyowrds, you can encounter only existing declarations on the left-hand side of an assigment.
Using `_ = ...` syntax may create impression that `_` is some existing global variable, while in reality, any expression that uses `_` — such as `println(_)` — would produce 'unresolved reference' compiler error.

## References and further extensions

Kotlin already supports underscores in several different positions:

* Underscore for unused lambda/anonymous function parameters ([KEEP](underscore-for-unused-parameters.md), [link](https://kotlinlang.org/docs/lambdas.html#underscore-for-unused-variables)).
* Underscore for unused components in positional-based destructuring ([link](https://kotlinlang.org/docs/destructuring-declarations.html)).
* Underscore for unused type arguments ([link](https://kotlinlang.org/docs/generics.html#underscore-operator-for-type-arguments))
* Underscore for unused exception instances ([link](https://youtrack.jetbrains.com/issue/KT-31567))

With this proposal, we would also support underscores for unused local variables.

Besides this list, one of the most requested places for underscore support is an unused function parameter:

```kotlin
fun foo(_: String) {
  println("Parameter is unused")
}
```

However, this use case is explicitly out of scope for this proposal.
Its destiny will be decided later.