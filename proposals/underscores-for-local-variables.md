# Underscore syntax for unused local variables

* **Type**: Design proposal
* **Authors**: Leonid Startsev, Mikhail Zarechenskiy
* **Contributors**: Alejandro Serrano Mena, Marat Akhin, Nikita Bobko, Pavel Kunyavskiy
* **Status**: Experimental in Kotlin 2.2
* **Discussion**: [link to discussion thread or issue]
* **Prototype**: [link to prototype or implementation, if any]

## Synopsis

Allow to use `_` (one underscore) as a local variable name to explicitly express that it is unused:

```kotlin
fun writeTo(file: File): Boolean {
    val result = runCatching { file.writeText("OK") }
    return result.isSuccess
}

fun foo(file: File) {
    val _ = writeTo(file)
}
```

Underscore can be used only in *local variable declaration*: `val _ = ...` and cannot be referenced or resolved otherwise.

## Motivation

> ChatGPT slop
1. Readiness for unused return value checker

The use of underscores for local variables provides readiness for unused return value checkers by explicitly marking values that are not intended to be used. This helps static analyzers identify intentional discards and avoid false positives, enabling developers to focus on genuinely problematic cases.

2. Better readability

Underscores improve the readability of code by clearly indicating that certain variables are irrelevant or disposable in a given context. This practice makes the intent of the code more obvious to readers and ensures that the focus remains on the meaningful portions of the logic.


In Kotlin, if you need to ignore or discard a function's return value intentionally, it’s important to clearly indicate this in your code for better readability and maintainability. Using an underscore `_` as a placeholder variable signals that the return value is not needed. This practice avoids potential confusion and makes your intention explicit to others reviewing the code, ensuring they understand you’re not accidentally ignoring something important. Explicitly discarding return values contributes to safer code and aligns with Kotlin's principle of making intent clear and reducing ambiguity.


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

With this proposal, Kotlin would support:

* Underscore for unused lambda/anonymous function parameters ([KEEP](underscore-for-unused-parameters.md), [link](https://kotlinlang.org/docs/lambdas.html#underscore-for-unused-variables)).
* Underscore for unused components in positional-based destructuring ([link](https://kotlinlang.org/docs/destructuring-declarations.html)).
* Underscore for unused type arguments ([link](https://kotlinlang.org/docs/generics.html#underscore-operator-for-type-arguments))
* Underscore for unused exception instances ([link](https://youtrack.jetbrains.com/issue/KT-31567))
* Underscore for unused local variables (this proposal)

Besides this list, one of the most requested places for underscore support is an unused function parameter:

```kotlin
fun foo(_: String) {
  println("Parameter is unused")
}
```

However, this use case is explicitly out of scope for this proposal.
Its destiny will be decided later.