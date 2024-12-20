# Collection Literals

* **Type**: Design proposal
* **Author**: Nikita Bobko
* **Contributors**: Alejandro Serrano Mena, Denis Zharkov, Marat Akhin, Mikhail Zarechenskii, todo
* **Issue:** [KT-43871](https://youtrack.jetbrains.com/issue/KT-43871)
* **Prototype:** https://github.com/JetBrains/kotlin/tree/bobko/collection-literals
* **Discussion**: todo

Collection literal is a well-known feature among modern programming languages.
It's a syntactic sugar built-in into the language that allows creating collection instances more concisely and effortlessly.
In simpliest form, if users want to create a collection, instead of writing `val x = listOf(1, 2, 3)` they could write `val x = [1, 2, 3]`.

## Table of contents

- [Motivation](#motivation)
- [Proposal](#proposal)
- [Overload resolution motivation](#overload-resolution-motivation)
  - [Overload resolution and type inference](#overload-resolution-and-type-inference)
  - [Operator function `of` restrictions](#operator-function-of-restrictions)
  - [Operator function `of` permissions](#operator-function-of-permissions)
  - [Theoretical possibility to support List vs Set overloads in the future](#theoretical-possibility-to-support-list-vs-set-overloads-in-the-future)
- [What happens if user forgets operator keyword](#what-happens-if-user-forgets-operator-keyword)
- [Similarities with `@OverloadResolutionByLambdaReturnType`](#similarities-with-overloadresolutionbylambdareturntype)
- [Feature interaction with `@OverloadResolutionByLambdaReturnType`](#feature-interaction-with-overloadresolutionbylambdareturntype)
- [Similar features in other languages](#similar-features-in-other-languages)
- [Interop with Java ecosystem](#interop-with-Java-ecosystem)
- [Tuples](#tuples)
- [Performance](#performance)
- [IDE support](#ide-support)
- [listOf deprecation](#listof-deprecation)
- [Change to stdlib](#change-to-stdlib)
  - [Semantic differences between Kotlin and Java factory methods](#semantic-differences-between-kotlin-and-java-factory-methods)
- [Empty collection literal](#empty-collection-literal)
- [Future evolution](#future-evolution)
- [Rejected proposals and ideas](#rejected-proposals-and-ideas)
  - [Rejected proposal: more granular operators](#rejected-proposal-more-granular-operators)
  - [Rejected idea: self-sufficient collection literals with defined type](#rejected-idea-self-sufficient-collection-literals-with-defined-type)
  - [Rejected proposal: use improved overload resolution algorithm only to refine non-empty overload candidates set on the fixated tower level](#rejected-proposal-use-improved-overload-resolution-algorithm-only-to-refine-non-empty-overload-candidates-set-on-the-fixated-tower-level)

## Motivation

1.  Marketing.
    A lot of modern languages have collection literals.
    The presence of this feature makes a good first impression on the language.
2.  Collections (not literals) are very widely used in programs.
    They deserve a separate literal.
    A special syntax for collection literal makes them instantly stand out from the rest of the program, making code easier to read.
3.  Simplify migration from other languages.
    Collection literals is a widely understood concept with more or less the same syntax across different languages.
    And new users have the right to naively believe that Kotlin supports it.
4.  Clear intend.
    A special syntax for collection literals makes it clear that a new instance consisting of the supplied elements is created.
    For example, `val x = listOf(10)` is potentially confusing, because some readers might think that a new collection with the capacity of 10 is created.
    Compare it to `val x = [10]`.
5.  Special syntax for collection literals helps to resolve the `emptyList`/`listOf` hassle.
    Whenever the argument list in `listOf` reduces down to zero, some might prefer to clean up the code to change `listOf` to `emptyList`.
    And vice versa, whenever the argument list in `emptyList` needs to grow above zero, the programmer needs to replace `emptyList` with `listOf`.
    It creates a small hussle of `listOf` to `emptyList` back and forth movement.
    It's by no means a big problem, but it is just a small annoyance, which is nice to see to be resolved by the introduction of collection literals.

The feature doesn't bring a lot of value to the existing users, and primarily targets newcomers.

## Proposal

**Definition.**
_`Type` static scope_ is the set that contains member callables (functions and properties) of `Type.Companion` type (`Type.Companion` is a companion object),
or static members of the type if the type is declared in Java.

It's proposed to use square brackets because the syntax is already used in Kotlin for array literals inside annotation constructor arguments,
because it's the syntax a lot of programmers are already familiar with coming from other programming languages,
and because it honors mathematical notation for matrices.

**Informally**, the proposal strives to make it possible for users to use collection literals syntax to express user-defined types `val foo: MyCustomList<Int> = [1, 2, 3]`.
And when the expected type is unspecified, expression must fall back to the `kotlin.List` type: `val foo = [1, 2, 3] // List<Int>`.

**More formally**, before the collection literal could be used at the use-site, an appropriate type needs to declare `operator fun of` function in its _static scope_.
The `operator fun of` functions must adhere to [the restrictions](#operator-function-of-restrictions).

Once proper `operator fun of` is declared, the collection literal can be used at the use-site.
1.  When the collection literal is used in the arguments position, similarly to lambdas and callable references, collection literal affects the overload resolution of the "outer call".
    See the section dedicated to [overload resolution](#overload-resolution-motivation).
2.  When the collection literal is used in the position with definite expected type, the collection literal is literally desugared to `Type.of(expr1, expr2, expr3)`,
    where `Type` is the definite-expected type.

    The following positions are considered positions with the definite expected type:
    - Conditions of `when` expression with a subject
    - `return`, both explicit and implicit
    - Equality checks (`==`, `!=`)
    - Assignments and initializations
3.  In all other cases, it's proposed to desugar collection literal to `List.of(expr1, expr2, expr3)`.

Some examples:
```kotlin
class MyCustomList<T> {
    companion object { operator fun <T> of(vararg t: T): MyCustomList<T> = TODO() }
}
fun main() {
    val foo: MyCustomList<Int> = [1, 2, 3]
    val bar: MyCustomList<String> = ["foo", "bar"]
    val baz = [1, 2, 3] // List<Int>
    when (foo) {
        [1, 2, 3] -> println("true")
        else -> println("false")
    }
    if (foo == [1, 2, 3]) println("true")
}

fun foo(): MyCustomList<Int> = [1, 2, 3]
```

Please note that it is not necessary for the type to extend any predifined type in the Kotlin stdlib (needed to support `kotlin.Array<T>` type),
nor it is necessary for the user-defined type to declare mandatory generic type parameters (needed to support specialized arrays like `kotlin.IntArray`, `kotlin.LongArray`, etc.).

## Overload resolution motivation

The reasoning behind [the restrictions](#operator-function-of-restrictions) on `operator fun of` are closely related to overload resolution and type inference.
Consider the following real-world example:
```kotlin
@JvmName("sumDouble") fun sum(set: List<Double>): Double = TODO("not implemented") // (1)
@JvmName("sumInt")    fun sum(set: List<Int>): Int = TODO("not implemented")       // (2)

fun main() {
    sum([1, 2, 3]) // Should resolve to (2)
}
```

We want to make the overload resolution work for such examples.
We have conducted an analysis to see what kinds of overloads are out there [KT-71574](https://youtrack.jetbrains.com/issue/KT-71574) (private link).
In short, there are all kinds of overloads.
The restrictions and the overload resolution algorithm suggested further will help us to make the resolution algorithm distinguish `List<Int>` vs `List<Double>` kinds of overloads.

> [!NOTE]
> When we say `List` vs `Set` kinds of overloads, we mean all sorts of overloads where the "outer" type is different.
> When we say `List<Int>` vs `List<Double>` kinds of overloads, we mean all sorts of overloads where the "inner" type is different.

-   `List<Int>` vs `Set<Int>` typically emerges when one of the overloads is the "main" overload, and another one is just a convenience overload that delegates to the "main" overload.
    Such overloads won't be supported because it's generally not possible to know which of the overloads is the "main" overload,
    and because collection literal syntax doesn't make it possible to syntactically distinguish `List` vs `Set`.
    But if we ever change our mind, [it's possible to support `List` vs `Set` kind of overloads](#theoretical-possibility-to-support-List-vs-Set-overloads-in-the-future) in the language in backwards compatible way in the future.
-   `List<Int>` vs `List<Double>` is self-explanatory (for example, consider the `sum` example above).
    Such overloads should be and will be supported.
-   `List<Int>` vs `Set<Double>`. Both "inner" and "outer" types are different. 
    This pattern is less clear and generally much rarer.
    The pattern may emerge accidentally when different overloads come from different packages.
    Or when users don't know about `@JvmName`, so they use different "outer" types to circumvent `CONFLICTING_JVM_DECLARATIONS`.
    This pattern will be supported just because of the general approach we are taking that distinguishes "inner" types.

### Overload resolution and type inference

> Readers of this section are encouraged to familiarize themselves with how [type inference](https://github.com/JetBrains/kotlin/blob/master/docs/fir/inference.md) and
> [overload resolution](https://kotlinlang.org/spec/overload-resolution.html#overload-resolution) already work in Kotlin.

All the problems with overload resolution and collection literals come down to examples when collection literals are used at the argument position.
```kotlin
fun <K> materialize(): K = null!!
fun outerCall(a: List<Int>) = Unit
fun outerCall(a: Set<Double>) = Unit

fun test() {
    outerCall([1, 2, materialize()])
}
```

On the high-level, overload resolution algorithm is very simple and logical.
There are two stages:
1.  Filter out all the overload candidates that certainly don't fit based on types of the arguments.
    Only types of non-lambda and non-reference arguments are considered.
    (it's important to understand that we don't keep the candidates that fit, but we filter out those that don't)
    https://kotlinlang.org/spec/overload-resolution.html#determining-function-applicability-for-a-specific-call
2.  Of the remaining candidates, we keep the most specific ones by comparing every two distinct overload candidates.
    https://kotlinlang.org/spec/overload-resolution.html#choosing-the-most-specific-candidate-from-the-overload-candidate-set

For our case to distinguish `List<Int>` vs `List<Double>` kinds of overloads (see the [Overload resolution motivation](#overload-resolution-motivation) section), the first stage of overload resolution is the most appropriate one.
That's exactly what we want to do - to filter out definitely inapplicable `outerCall` overloads.

For simplicity, imagine that there is only a single `operator fun of` declaration inside `companion object` of our target class.
And that single overload has single `vararg` parameter.
In the next section, you will see that `operator fun of` restrictions are such that overload resolution algorithm is no different for our simplified case and for a more complicated case when some limited `of` overloads are permitted.

Given the following example: `outerCall([expr1, [expr2], expr3, { a: Int -> }, ::x], expr4, expr5)`,
similar to lambdas and callable references, collection literal expression type inference is delayed.
Contrary, elements of the collection literal are analyzed in the way similar to how other arguments of `outerCall` are analyzed, which means:
1.  If collection literal elements are lambdas, or callable references, their analysis is delayed.
    Only number of lambda parameters and lambda parameter types (if specified) are taken into account of overload resolution of `outerCall`.
2.  If collection literal elements are collection literals themselves, then we descend into those literals and recursively apply the same rules.
3.  All other collection literal elements are "plain arguments", and they are analyzed in [so-called "dependent" mode](https://github.com/JetBrains/kotlin/blob/master/docs/fir/inference.md).

For every overload candidate, when a collection literal maps to its appropriate `ParameterType`:
1.  We find `ParameterType.Companion.of(vararg)` function.
    Remember that our simplified example is such that there is only one such function.
    And actually, `operator fun of` function restrictions in the next section guarantee us that it's the only such function.
2.  We remember the parameter of the single `vararg` parameter.
    We will call it _CLET_ (collection literal element type).
    We also remember the return type of that `ParameterType.of(vararg)` function.
    We will call it _CLT_ (collection literal type).
3.  In the first stage of overload resolution of `outerCall` (when we filter out inapplicable candidates), we add the following constraints to the constraint system of `outerCall` candidate:
    1. For each collection literal element `e`, we add `type(e) <: CLET` constraint.
    2. We also add the following constraint: `CLT <: ParameterType`.

Once all "plain" arguments are analyzed (their types are infered in "dependent" mode), and all recursive "plain" elements of collection literals are analyzed,
we proceed to filtering overload candidates for `outerCall`.

Please note that constraints described above are only added to the constraint system of the `outerCall` and not to constraint system of the `of` function themselves.
(overload resolution for which will be performed later, once the overload resolution for `outerCall` is done)

Example:
```kotlin
class List<T> { companion object {
    operator fun <T> of(vararg t: T): List<T> = TODO("not implemented")
} }

fun <K> materialize(): K = null!!
fun outerCall(a: List<Int>) = Unit // (1)
fun outerCall(a: List<String>) = Unit // (2)

fun test() {
    // The initial constraint system for candidate (1) looks like:
    //                type(1) <: T
    //                type(2) <: T
    // type(materialize<K>()) <: T
    //                List<T> <: List<Int>
    // The constraint system is sound => the candidate is not filtered out

    // The initial constraint system for candidate (2) looks like:
    //                type(1) <: T
    //                type(2) <: T
    // type(materialize<K>()) <: T
    //               List<T> <: List<String>
    // The constraint system is unsound => the candidate is filtered out
    outerCall([1, 2, materialize()]) // We resolve to the candidate (1)
}
```

It's important to understand that until we pick the `outerCall` overload, we don't start the full-blown overload resolution process for nested `of` calls.
If we did so, it'd lead to exponential complexity of `outerCall` overload resolution.
Consider the `outerCall([[[1, 2, 3]]])` example.

Once the particular overload for `outerCall` is chosen, we know what definite expected type collection literal maps to.
We desugar collection literal to `DefiniteExpectedType.of(expr1, expr2, expr3)`, and we proceed to resolve overloads of `DefiniteExpectedType.of` according to regular Kotlin overload resolution rules.

### Operator function `of` restrictions

**The overall goal of the restrictions:**
Make it possible to extract type restrictions for the elements of the collection literal without the necessity of the full-blown real overload resolution for `operator fun of` function.
Given only the "outer" type `List<Int>`/`IntArray`/`Foo`, we should be able to infer collection literal element types.
We need to know the types for the constraint system of the `outerCall` like in the following example:
```
@JvmName("sumDouble") fun outerCall(set: List<Double>): Double = TODO("not implemented") // (1)
@JvmName("sumInt")    fun outerCall(set: List<Int>): Int = TODO("not implemented")       // (2)

fun main() {
    outerCall([1, 2, 3]) // Should resolve to (2)
}
```

**Restriction 1.**
Extension `operator fun of` functions are forbidden.
All `operator fun of` functions must be declared as member functions of the target type Companion object.
```kotlin
class Foo { companion object }
operator fun Foo.Companion.of(vararg x: Int) = Foo() // Forbidden
```

It's a technical restriction driven by the fact that, in Kotlin, extensions can win over members if members are not applicable.
The presence of extensions makes it impossible to check all further restrictions.

One could argue that it's already possible for all other `operator`s in Kotlin to be declared as extensions rather than members,
and it feels limiting not being possible to do the same for `operator fun of`.
Formally, `operator fun of` is different.
All `operator`s in Kotlin operate on the existing expression of the target type, while `operator fun of` doesn't have access to the expression of its target type, `operator fun of` is the constructor of its target type.
We could even replace `operator` keyword with another keyword to make it clearer that `operator fun of` is not a regular operator, but we don't see a lot of practical value in it.

**Restriction 2.**
One and only one overload with a single `vararg` parameter must be declared.
No more, no less.

The overload is considered the "main" overload, and it's the overload we use to extract type constraints from for the means of `outerCall` overload resolution.
Please remember that we treat collection literal argument as a "delayed argument" (similar to lambdas and callable references).
First, We do the overload resolution for the `outerCall` and only once a single applicable candidate is found, we use its appropriate parameter type as an expected type for the collection literal argument expression.

Correct:
```kotlin
class Foo {
    companion object {
        operator fun of(vararg x: Int): Foo = Foo()
        operator fun of(x: Int): Foo = Foo()
        operator fun of(x: Int, y: Int): Foo = Foo()
    }
}
```

Incorrect:
```kotlin
class Foo {
    companion object {
        operator fun of(vararg x: Int): Foo = Foo()
        operator fun of(vararg x: String): Foo = Foo() // Duplicated `vararg` overload
    }
}
```

Also, incorrect:
```kotlin
class Pair {
    companion object {
        operator fun of(x: Int, y: Int): Pair = Pair() // `vararg` overload is not declared
    }
}
```

**Restriction 3.**
All `of` overloads must have the return type equal by `ClassId` to the type in which _static scope_ the overload is declared in.

Since the class in which the static scope we search `of` function in is selected by the expected type,
the expected type should match with the type of the expression (expression = collection literal).
Otherwise, it's just nonsense.

Supportive example:
```kotlin
class Foo { companion object {
    operator fun of(vararg x: Int): String = ""
} }

fun main() {
    val x: Foo = [1] // [TYPE_MISMATCH] expected: Foo, got: String
}
```

**Restriction 4.**
All `of` overloads must have the same return type.

Supportive example:
```kotlin
class MyList<T> { companion object {
    operator fun of(vararg x: Int): MyList<Int> = MyList()
    operator fun of(x: Int, y: Int): MyList<String> = MyList() // not allowed
} }

fun outerCall(a: MyList<Int>) = Unit // (1)
fun outerCall(b: List<Int>) = Unit // (2)

fun main() {
    // Since the restrictions for `outerCall` overload resolution are taken from the `vararg` overload,
    // both (1) and (2) are applicable and the call results in OVERLOAD_RESOLUTION_AMBIGUITY
    outerCall([1, 2])
    // But the following call resolves to (2)
    outerCall(MyList.of(1, 2))
}
```

**Restriction 5.**
The only difference that `of` overloads are allowed to have is "number" of parameters (`vararg` is considered an infinite number of params).

> Technically, restriction 5 is a superset of restriction 4, but we still prefer to mention restriction 4 separately.

Which means that types of the parameters must be the same, and all type parameters with their bounds must be the same.

We acknowledge that this restriction disallows having collection literals for things like `NonEmptyList` that may want to declare the following operator: `operator fun <T> of(first: T, vararg rest: T)`.

**Restriction 6.**
All `of` overloads must have zero extension/context parameters/receivers.

We forbid them to keep mental model simpler and since we didn't find major use cases.
Since all those "implicit receivers" affect availability of the `of` function, it'd complicate `outerCall` overload resolution, if we allowed "implicit receivers".

### Operator function `of` permissions

We would like to explicitly note the following permissions.

**Permission 1.**
It's allowed to have reified generics and mark the operator as `inline`.

Use case:
```kotlin
class IntArray {
    companion object {
        operator fun inline <reified T> of(vararg elements: T): IntArray /*the implementation is code generated*/
    }
}
```

**Permission 2.**
The operator is allowed to be `suspend`, or `tailrec`.
Because all other operators are allowed being such as well, and we don't see reasons to restrict `operator fun of`.

**Permission 3.**
It's allowed to have overloads that differ in number of arguments:
```kotlin
class List<T> {
    companion object {
        operator fun <T> of(vararg elements: T): List<T> = TODO()
        operator fun <T> of(element: T): List<T> = TODO()
        operator fun <T> of(): List<T> = TODO()
    }
}
```

**Permission 4.**
Java static `of` member is perceived as `operator fun of` if it satisfies the above restrictions.

### Theoretical possibility to support List vs. Set overloads in the future

We don't plan to, but if we ever change our mind, it's possible to support `List` vs `Set` kinds of overloads in the way similar to how Kotlin prefers `Int` overload over `Long` overload:
```kotlin
fun foo(a: Int) = Unit // (1)
fun foo(a: Long) = Unit // (2)

fun test() {
    foo(1) // (1)
}
```

On the second stage of overload resolution, `Int` is considered more specific than `Long`, `Short`, `Byte`.
In the similar way, `List` can be theoretically made more specific than any other type that can represent collection literals.

But right now, we **don't plan** to do that, since both `List` and `Set` overloads can equally represent the "main" overload.

## Similarities with `@OverloadResolutionByLambdaReturnType`

The suggested algorithm of overload resolution for collection literals shares similarities with `@OverloadResolutionByLambdaReturnType`.

Similar to how "the guts" of the lambda (the type of the return expression) are analyzed for the sake of the `outerCall` overload resolution,
"the guts" of collection literal (elements of the collection literal) are analyzed for the same purpose.
The big difference though is that analysis of "the guts" of collection literal doesn't depend on some "input types" coming from the signature of the particular `outerCall`,
while in the case of lambdas it's different.
You need to know the types of the lambda parameters (so-called "input types") to infer the return type of the lambda.

That's why in the case of collection literals, we can jump right into the analysis of its elements, and only postpone the overload resolution of the `operator fun of` function.

Another big difference is what stage the improved overload resolution by lambda return type or by collection literal element type kicks in.

Improved overload resolution by collection literal element type naturally merges itself into the first stage of overload resolution (candidates filtering), where it logically belongs to.
Contrary, `@OverloadResolutionByLambdaReturnType` is a separate stage of overload resolution that kicks in after choosing the most specific candidate.

The fact that `@OverloadResolutionByLambdaReturnType` is just slapped on top of regular overload resolution algorithm can be observed in the following example:
```kotlin
interface Base
object A : Base
object B : Base

@OptIn(kotlin.experimental.ExperimentalTypeInference::class)
@OverloadResolutionByLambdaReturnType
@JvmName("mySumOf2")
fun mySumOf(body: () -> Base) = Unit // (1)

@OptIn(kotlin.experimental.ExperimentalTypeInference::class)
@OverloadResolutionByLambdaReturnType
@JvmName("mySumOf3")
fun mySumOf(body: () -> B) = Unit // (2)

fun main() {
    mySumOf({ A }) // Actual: resolve to (2). ARGUMENT_TYPE_MISMATCH. Actual type is A, but B was expected
                   // Expected: resolve to (1). Green code
}
```

Collection literals don't suffer from this problem.

> The section is written for the sake of increasing the understanding of the mental model

## Feature interaction with `@OverloadResolutionByLambdaReturnType`

```kotlin
@OptIn(kotlin.experimental.ExperimentalTypeInference::class)
@OverloadResolutionByLambdaReturnType
@JvmName("mySumOf1")
fun mySumOf(body: List<() -> Int>) = Unit // (1)

@OptIn(kotlin.experimental.ExperimentalTypeInference::class)
@OverloadResolutionByLambdaReturnType
@JvmName("mySumOf2")
fun mySumOf(body: List<() -> Long>) = Unit // (2)

fun main() {
    mySumOf([{ 1L }])
}
```

Technically, since collection literal elements are analyzed "like regular arguments",
`@OverloadResolutionByLambdaReturnType` in the above case could make the `mySumOf` to resolve to (2).

We should make sure that the example above either results in `OVERLOAD_RESOLUTION_AMBIGUITY` or is prohibited in some way
(though it's unclear how to prohibit it)

## Similar features in other languages

**Java.**
Java explicitly voted against collection literals in favor of `of` factory methods.

> However, as is often the case with language features, no feature is as simple or as clean as one might first imagine, and so collection literals will not be appearing in the next version of Java.

[JEP 269: Convenience Factory Methods for Collections](https://openjdk.org/jeps/269).
[JEP 186: Collection Literals](https://openjdk.org/jeps/186).

**Swift.**

Kotlin proposal for collection literals shares a lot of similarities with the way collection literals are done in Swift.

Swift supports collection literals with the similar suggested syntax.
Swift also allows 

todo

## Interop with Java ecosystem

The name for the `operator fun of` is specifically chosen such to ensure smooth Java interop.
Given that we don't support extension `operator fun of`, it becomes more important for Java developers to declare `of` members that satisfy the requirements.
We hope that the JVM ecosystem will follow the "Convenience factory methods" pattern that Java started.
For example, one can already find convenience factory "of" methods in popular Java libraries such as Guava.

We perceive Java static `of` function as an `operator fun of` function only if it follows the restrictions mentioned above.
All the restrictions are reasonable, and we think that all "collection builder like" of functions will naturally follow those restrictions.

## Tuples

With the addition of collection literals, some users might want to use square brackets syntax to express tuples (well, Kotlin doesn't have tuples, but there are `kotlin.Pair` and `kotlin.Triple`).
The restrictions that we put on the `operator fun of` function don't make it possible to express tuples in a type-safe manner (user has to declare an `of(vararg)` overload).

We don't plan to support the "tuples use-case" in the first version of collection literals.
But in the future, it's yet unclear if we want to make tuples expressible via square brackets or maybe some other syntax.
So for now, we just want to make sure that we don't accidentally make it impossible to re-use square brackets syntax for tuples.

## Performance

We did performance measurements to make sure that we don't miss any obvious problems in Kotlin's `listOf` implementation compared to `java.util.List.of`.
And to compare performance of the proposal with [the alternative "granular operators" suggestion](#rejected-proposal-more-granular-operators), since the more granular operators proposal came as an idea to improve performance.
Unironically, a more straightforward "single operator of" proposal shows better performance than "granular operators proposal" in our benchmarks.

For array-backed collections, the approach to prepoluate array and pass the reference to the array is more performant than consecutively calling `.add` for every element.
Our benchmarks show 3x better performance.

Though, it's correct that for non-array-backed collections (like Sets or Maps), consecutive `.add`/`.put` are more performant since the `vararg`-allocated array is redundant.
But the performance boost of "granular operators proposal" for Maps and Sets is less significant (only 2x only for Maps and primarily because of `kotlin.Pair`/`Map.Entry` boxing, not because of unnecessary array allocation) compared to the performance degradation it causes to Lists (3x as it's already been mentioned).
Taking into account that List is the most popular collection container, the choice of the operator convention becomes obvious.

It's worth mentioning that the design decision shouldn't be driven purely by performance.
In our case, a more accurate design proposal just happens to be more performant than the suggested more granular alternative.
It's a double win situation.

**"Unique vararg" statement.** Unlike in Java, in _pure_ Kotlin, we can practically assume that `vararg` parameter is a unique array reference.

**Informal proof.** Given the following function declaration: `fun acceptVararg(vararg x: Int) = Unit`, let's consider the following cases.

-   Case 1. Pass elements of the `vararg` as separate arguments. It's obvious that a new unique array is created at the call-site.
-   Case 2. Use spread operator to "spread" existing array like this: `acceptVararg(*anotherExistingArray)`. Spread operator creates the array on the call-site.
-   Case 3. Bypass through callable reference like this: `::acceptVararg.invoke(intArrayOf(1, 2))`.
    Indeed, in this case `vararg` parameter cannot be guaranteed to be unique, since the argument is not copied on the call-site.

    Luckily for us, we don't consider this example practical, people don't create references to `listOf`, `List.of`, etc. functions.
    The primary usage for those functions is to call them directly with parameters mapped to the `vararg` parameter like this: `listOf(1, 2, 3)`.

So unless the Kotlin declaration that accepts `vararg` is called from Java, we can practically assume that the `vararg` parameter is unique.

Java doesn't assume that the vararg array is unique, so they have to copy it.
And to reduce the array copying performance impact, Java declares 10 `List.of` overloads, in which they don't need to do the copy.
Unnecessary array copying can cause more than 2x performance degradation.
Thanks to _"Unique vararg" statement_, Kotlin doesn't have to declare 10 `List.of` overloads like Java did.
The proposal is to proceed with just three overloads as we have them right now in Kotlin.

All the mentioned numbers can be seen in the raw benchmark data, that can be found in the [`resources/collection-literals-benchmark/`](https://github.com/Kotlin/KEEP/blob/master/resources/collection-literals-benchmark) directory of the KEEP repo root.

## IDE support

The IDE should implement an inspection to replace `listOf`/`setOf`/etc. with collection literals where it doesn't lead to overload resolution ambiguity.
It's under the question if the inspection for `listOf`/`setOf` should be enabled by default. (most probably not)

The IDE should implement an inspection to replace explicit use of operator function `Type.of` with collection literals where it doesn't lead to overload resolution ambiguity.
The inspection should be enabled by default.

We should take into account that ctrl-click on a single character is tricky, especially for via keyboard shortcuts.
Though there are some suggestions to improve the situation: [KTIJ-28500](https://youtrack.jetbrains.com/issue/KTIJ-28500).

## listOf deprecation

We **don't** plan to deprecate any of the `smthOf` functions in Kotlin stdlib.
They are too much widespread, even some third party Kotlin libraries follow `smthOf` pattern.

Besides, this proposal doesn't allow to eliminate `smthOf` pattern completely.
`listOfNotNull` isn't going anywhere.

Unfortunately, introduction of `List.of` functions in stdlib makes the situation harder for newcomers,
since they will have more ways to create collections `listOf`, `List.of`, and `[]`.

For the reference, here is the full list of `smthOf` functions that we managed to find in stdlib:

List like
- `listOf()`
- `listOfNotNull()`
- `setOfNotNull()`
- `mutableListOf()`
- `sequenceOf()`
- `arrayListOf()`
- `arrayOf()`
- `intArrayOf()`, `shortArrayOf()`, `byteArrayOf()`, `longArrayOf()`, `ushortArrayOf()` // etc
- `setOf()`
- `mutableSetOf()`
- `hashSetOf()`
- `linkedSetOf()`
- `sortedSetOf()`

Map like
- `mapOf()`
- `mutableMapOf()`
- `hashMapOf()`
- `sortedMapOf()`

Other collections
- `arrayOfNulls()`
- `arrayOf().copyOf()` // vs. spread in collection literals?
- `mutableListOf<String>().copyOf()` // red code. We don't have a stdlib function to copy list in Kotlin

Others, not collections, but things that could be literals of their own
- `lazyOf()`
- `typeOf<String>()` // Defined in kotlin-reflect. returns KType
- `enumValueOf<E>(String)`
- `CharDirectionality.valueOf(Int)` // Int to Enum entry
- `java.lang.Boolean.valueOf(true)` // Java
- `java.lang.Byte.valueOf("")` // Java
- `java.math.BigInteger.valueOf(1l)` // Java

## Change to stdlib

The following API change is proposed to stdlib:
```diff
diff --git a/libraries/stdlib/src/kotlin/Collections.kt b/libraries/stdlib/src/kotlin/Collections.kt
index e692a8c05ede..7509cc55b9c6 100644
--- a/libraries/stdlib/src/kotlin/Collections.kt
+++ b/libraries/stdlib/src/kotlin/Collections.kt
@@ -175,6 +175,13 @@ public expect interface List<out E> : Collection<E> {
      * Structural changes in the base list make the behavior of the view undefined.
      */
     public fun subList(fromIndex: Int, toIndex: Int): List<E>
+
+    public companion object {
+        public operator fun <T> of(vararg elements: T): List<T>
+        public operator fun <T> of(element: T): List<T>
+        public operator fun <T> of(): List<T>
+    }
 }

 /**
```

The analogical API changes are proposed in the following classes:
- `kotlin.collections.ArrayList`
- `kotlin.collections.MutableList`
- `kotlin.collections.Set`
- `kotlin.collections.HashSet`
- `kotlin.collections.LinkedHashSet`
- `kotlin.collections.MutableSet`
- `kotlin.sequences.Sequence`
- `kotlin.Array`, `kotlin.IntArray`, `kotlin.LongArray`, `kotlin.ShortArray`, `kotlin.UIntArray` etc.

Please note that on JVM, `kotlin.collections.List` is a mapped type.
Which means that we cannot change the runtime `java.util.List`.
It's suggested to do mapped `companion object` in the way similar to `Int.Companion` (which maps to `kotlin.jvm.internal.IntCompanionObject`).

### Semantic differences between Kotlin and Java factory methods

- In Java, `java.util.List.of(null)` throws NPE.
- In Kotlin, `listOf(null)` returns `List<Nothing?>`.

Given that Kotlin has nullability built-in, it's proposed for `kotlin.collections.List.of(null)` to work similarly to `listOf(null)`.

- In Java, duplicated elements in `java.util.Set.of(1, 1)` cause `IllegalArgumentException`.
- In Kotlin, `setOf(1, 1)` returns a Set that consists of a single element.

Since it might be very unexpected for expression `Set.of(compute(), compute())` to throw an exception, it's proposed for `kotlin.collections.Set.of(1, 1)` to work similarly to `setOf(1, 1)`.

Both Java and Kotlin return unmodifiable collections.
Though, in Kotlin, we found a "problem" that `mapOf(vararg)` and `setOf(vararg)` don't do that.
The reasons for that are yet unknown, it could be an oversight, or it could be a deliberate choice because `Collections.unmodifiableMap` can be slower, since it needs to create `UnmodifiableEntry` for every element.

## Empty collection literal

`emptyList` stdlib function declares `List<T>` as its return type, not `List<Nothing>`.
Similar to `emptyList`, if the expected type doesn't provide enough information for what the collection literal element should be,
Kotlin compiler should issue a compilation error asking for an explicit type specification.

```kotlin
fun main() {
    val list = [] // Compilation error. Can't infer List<T> generic parameter T
}
```

## Future evolution

In the future, Kotlin may provide `@VarargOverloads` (similar to `@JvmOverloads`), or `inline vararg` to eliminate unnecessary array allocations even further.
But it's relevant only for Maps and Sets.

## Rejected proposals and ideas

This section lists some common proposals and ideas that were rejected

### Rejected proposal: more granular operators

In [the proposal](#proposal), we give users possibility to add overloads for the `vararg` operator.
It's clear that we do so to avoid unnecessary array allocations at least for most popular cases when collection sizes are small.
What if instead of a single operator, collections had to declare three operators: `createCollectionBuilder`, `plusAssign`, `freeze` (we even already have the `plusAssign` operator!)?

Please note that we need the third `freeze` operator to make it possible to create collection literals for immutable collections.

Then the use-site could be desugared by the compiler:
```kotlin
fun main() {
    val s = [1, 2, 3]
    // desugared to
    val s = run {
        val tmp = createCollectionBuilder<Int>()
        tmp.plusAssign(1)
        tmp.plusAssign(2)
        tmp.plusAssign(2)
        List.freeze(tmp)
    }
}
```

The declaration site looks like:
```kotlin
class List<T> {
    companion object {
        operator fun createCollectionBuilder<T>(capacity: Int): MutableList<T> = ArrayList(capacity)
        operator fun <T> freeze(a: MutableList<T>): List<T> = a.toList()
    }
}

class MutableList<T> {
    fun add(t: T): Boolean = TODO()

    companion object {
        operator fun createCollectionBuilder<T>(capacity: Int): MutableList<T> = ArrayList(capacity)
        operator fun <T> freeze(a: MutableList<T>): MutableList<T> = this
    }
}

operator fun <T> MutableList<T>.plusAssign(t: T) { add(t) }
```

There are several problems.
1.  The design with 3 operators is simply more complicated.
    3 operators become more scattered than a single operator, it becomes unclear where the IDE should navigate on ctrl-click.
    So if possible, we should prefer a more straightforward "single operator" design.
2.  The type of the collection builder is exposed in public API (return type of `createCollectionBuilder`).
    The type of the builder is an implementation detail, and users might want to change it over time.
3.  It becomes harder or unreasonable to return different collection types/builder types depending on the number of elements in the collection literal.
4.  Performance.
    For array-backed collections, the approach to prepopulate an array and safe the reference to the array is more performant than calling `.add` for every element.
    The most popular collection is List.
    List is array-backed.
    Though it's correct that consecutive `.add` calls is more performant for non-array-backed collections like Map or Set.
    Please see the [Performance](#performance) section.

### Rejected idea: self-sufficient collection literals with defined type

> todo it's not yet completely rejected since the discussion around Map literals is still in progress

The type of collection literals is infered from the expected type.
Unfortunately, that may lead to overload resolution ambiguity when collection literal is used in argument position.

```kotlin
fun foo(a: Set<Int>) = Unit
fun foo(a: List<Int>) = Unit

fun test() {
    foo([1, 2]) // overload resolution ambiguity
    // but what if...
    foo(List [1, 2])
}
```

The rejected proposal was to allow writing `List [1, 2]`.
The proposal was rejected because users can anyway desugar `[1, 2]` to `List.of(1, 2)` or `listOf(1, 2)`.

### Rejected proposal: use improved overload resolution algorithm only to refine non-empty overload candidates set on the fixated tower level

**Fact.** `foo` in the following code resolves to (2). Although (1) is a more specific overload, (2) is preferred because it's more local.

```kotlin
fun foo(a: Int) {} // (1)
class Foo {
    fun foo(a: Any) {} // (2)
    fun test() {
        foo(1) // Resolves to (2)
    }
}
```

It was proposed to use an improved overload resolution algorithm for collection literals to refine already non-empty overload candidates set to avoid resolving to functions which are "too far away".

```kotlin
fun foo(a: List<Int>) {} // (1)
class Foo {
    fun foo(a: List<String>) {} // (2)
    fun test() {
        foo([1]) // It's proposed to resolve to (2) and fail with TYPE_MISMATCH error
    }
}
```

The proposal was rejected because the analogical "improved overload resolution for references" already allows resolving to functions which are "too far away".

```kotlin
fun x(a: Int) = Unit
fun foo(a: (Int) -> Unit) {} // (1)
class Foo {
    fun foo(a: (String) -> Unit) {} // (2)
    fun test() {
        foo(::x) // Resolves to (1)
    }
}
```

We prefer to keep the language consistent rather than "fixing" the behavior, even if the suggested behavior is generally better (which is not clear, if it's really better)
