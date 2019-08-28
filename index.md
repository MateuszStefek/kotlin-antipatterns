This is a list of Kotlin-specific antipatterns.
Naturally, it focuses on the abuse of the features of the language. 

# 1. Elvis returns null
## Description
Normally, you handle undesired nulls by throwing an exception: 

```
fun dogHello(): String {
  val dog = selectDog() ?: throw IllegalStateException("no dog")
  return dog.bark()
}
```

It is tempting, however, to simply return null, and propagate it through the layers of the program.
The elvis operator - `?:` makes it extremely easy.
Simply add `?: return null` everywhere the compiler complains about nullability:
```
fun dogHello(): String? {
  val dog = selectDog() ?: return null
  return dog.bark()
}
```  
```
fun helloWorld(): List<String>? {
  return listOf(
    dogHello() ?: return null,
    catHello() ?: return null
  )
}
```
```
fun main() {
  helloWorld()?.forEach { println(it) } 
}
```

## Why does it smell?
- It's a surprising way of dealing with errors.

   Surprises are not welcome.

- `null` carries less information from what you can put into an exception object.

   In the future, when this code needs to be refactored to distinguish between different
   types of problems, you will need to modify a lot of lines.

- `?` explosion

   Frivoulous `?` tend to proliferate across the code base.
   Having multiple nullable variables defeats the main advantage of Kotlin.

## Solution
Use exceptions - this is what they are designed for.

One might argue that since Kotlin doesn't have checked exceptions,
there's no way to force the users of your functions to remember about error handling.
The compiler-forced null checks can be used as a cheap substitution 
for compiler-forced exception handling.
Don't be lazy - there are better alternatives.
For example, you can implement a version of Either.
It is a shame that Kotlin std library doesn't have this structure already.

# 2. Also this is null
Credits for naming this antipattern goes to
[Paul Blundell](https://blog.novoda.com/kotlin-anti-patterns-also-this-is-ull/).
## Description
```
val dog: Dog? = selectDog()

dog?.also {
  it.doBark()
  it.doSit()
}
```
The code inside the lambda will be executed only if `dog` is not null.
## Why does it smell?
- This expression has the same structure as the good old `if (dog != null) {`.

   If there are multiple ways of writing the same thing,
   choose the syntax which is more natural to most programmers.
   Newcomers from Java world will thank you.
   
- There's a micro-scope with the `it` variable.

   It takes a few seconds to be compiled in some brains.
   
- It looks like a functional code, but actually is a sequence of imperative commands.
## Solution

Use `if` whenever it's possible:
```
if (dog != null) {
  dog.doBark()
  dog.doSit()
}
```
Remember that `if ` and `when` can also be used as expressions.

I have to admit, that sometimes this structure is helpful when you really don't want to
introduce a new variable:
```
a.b.c?.also {
  it.d()
  it.e()
}
```
However, don't be fooled - you are actually introducing a new variable.

It is called `it` and it is not a good name for a variable as 
it - the name of the `it` variable is too generic.

# 3. Coalesce train ?.?.?.?.

## Description
```
a()?.b()?.c()?.d()
```
## Why does it smell?

Long chains of calls are suspicious [in general](https://en.wikipedia.org/wiki/Law_of_Demeter).
The `?.` operator allows you to write chains of calls in places where you
would have to stop and think, if you had to write the same code in Java.
Just because it's possible, doesn't mean it is a good idea.

Another problem with chained `?.` is that the operator is left-associative:
```
a()?.b()?.c()?.d()
```
is the same as:
```
((a()?.b())?.c())?.d()
```
Even if only the first call returns a nullable expression, the whole chain is infected with nullability.
This makes it harder to understand what causes the null in the chain.

## Solution

Avoid long chain of calls - name your stuff.

If you can't resist, you can use this trick to at least contain the nullability:
```
a()?.run { b().c().d() } 
```


# 4. Code golfing with scope functions
## Description
Suppose you have a `List<Pair<String, Int?>>`.
You want to filter out the pairs with nulls, in such a way that the resulting expression
is of type `List<Pair<String, Int>>`.

This is a common problem, because such lists of pairs are often passed as arguments of `mapOf()`.
A typical solution on [Stackoverflow](https://stackoverflow.com/a/49347526/4076606) suggests this:

```
val map = listOf(Pair("a", 1), Pair("b", null), Pair("c", 3), Pair("d", null))
    .mapNotNull { p -> p.second?.let { Pair(p.first, it) } }
    .toMap()
```

The answer is correct.
The type of the `map` variable is inferred to `Map<String, Int>`.

We can go further and "simplify" it even more:

```
val map = listOf(Pair("a", 1), Pair("b", null), Pair("c", 3), Pair("d", null))
            .mapNotNull { with(it) { second?.run { first to this } } }
            .toMap()
```

## Why does it smell?

How long did it take you to understand why the resulting map has non-nullable keys?

The first time I saw this trick,
I had to pause for more than a minute to mentally parse and simulate the types.

Such trivial manipulations should not bother the reader.
If pairs with null values are filtered out, the code should literally
spell something like `.withoutNullSecondComponent()`.

## Solution
Remember, the code is written once and read multiple times.
Some readers don't have time to parse your clever tricks
and some of them will be rightfully annoyed.

Introduce helper functions.
Kotlin makes it natural with extensions:

```
fun <K, T: Any> Map<K, T?>.filterEntriesWithNullValues(): Map<K, T> {
```

Now, you can put as many clever tricks as you want into the body of this function.
The implementation will not be inspected by readers,
because the name and the signature are descriptive enough.