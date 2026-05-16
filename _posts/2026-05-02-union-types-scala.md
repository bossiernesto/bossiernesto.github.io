---
layout: posts
title: "Union type in Scala"
excerpt: "Union Types in Scala 2 and 3"
image:
  path: 
categories:
  - Union Types
  - Scala
  - Functional Programming
tags:
  - functional
  - computer science
---


# Folding Union Types in Scala 2 and Scala 3

## Introduction

Union types represent values that may belong to one among multiple possible types. While this feature is built directly into Scala 3 through the `|` operator, Scala 2 required developers to rely on advanced type-level encodings in order to approximate similar behavior.

One of the most well-known approaches used logical encodings inspired by the Curry–Howard correspondence, allowing union-like constraints to be represented through negation and double negation. Although these techniques were never native language features, they demonstrated the expressive power of Scala’s type system and became a notable example of type-level programming in Scala 2.

This article presents a `foldUnion` implementation for both Scala 2 and Scala 3. The Scala 2 version uses a classical union encoding combined with runtime dispatch, while the Scala 3 version illustrates how native union types simplify the same idea considerably.

---

## Union Type Encoding in Scala 2

A common Scala 2 encoding for union types is the following:

```scala
type ¬[A] = A => Nothing
type ∨[T, U] = ¬[¬[T] with ¬[U]]
type ¬¬[A] = ¬[¬[A]]

type |∨|[T, U] = {
  type λ[X] = ¬¬[X] <:< (T ∨ U)
}
```

This encoding relies on logical negation and double negation in order to express type-level disjunction.

Using this construction, we can constrain a type parameter as follows:

```scala
def size[T: (Int |∨| String)#λ](t: T): Int = ???
```

Conceptually, this expresses that `T` must belong to the union `Int ∨ String`.

Although this is not a native union type, it provides a useful compile-time restriction and enables APIs that resemble union-based programming.

---

## An Improved `foldUnion` for Scala 2

The following implementation modernizes the original approach by using `ClassTag` instead of `ClassManifest`, handling primitive boxing explicitly, and providing clearer runtime errors.

```scala
import scala.reflect.ClassTag

type ¬[A] = A => Nothing
type ∨[T, U] = ¬[¬[T] with ¬[U]]
type ¬¬[A] = ¬[¬[A]]

type |∨|[T, U] = {
  type λ[X] = ¬¬[X] <:< (T ∨ U)
}

final class FoldUnionOps[T](private val value: T) extends AnyVal {

  private def boxed(c: Class[_]): Class[_] =
    if (!c.isPrimitive) c
    else if (c == java.lang.Integer.TYPE) classOf[java.lang.Integer]
    else if (c == java.lang.Long.TYPE) classOf[java.lang.Long]
    else if (c == java.lang.Double.TYPE) classOf[java.lang.Double]
    else if (c == java.lang.Float.TYPE) classOf[java.lang.Float]
    else if (c == java.lang.Short.TYPE) classOf[java.lang.Short]
    else if (c == java.lang.Byte.TYPE) classOf[java.lang.Byte]
    else if (c == java.lang.Character.TYPE) classOf[java.lang.Character]
    else if (c == java.lang.Boolean.TYPE) classOf[java.lang.Boolean]
    else if (c == java.lang.Void.TYPE) classOf[java.lang.Void]
    else c

  private def isSubtypeOf(actual: Class[_], expected: Class[_]): Boolean =
    expected.isAssignableFrom(actual) ||
      boxed(expected).isAssignableFrom(boxed(actual))

  def foldUnion[A, B, S](
    fa: A => S,
    fb: B => S
  )(implicit
    ev: ¬¬[T] <:< (A ∨ B),
    ca: ClassTag[A],
    cb: ClassTag[B]
  ): S = {

    if (value == null) {
      throw new MatchError("Cannot fold null as a union value")
    }

    val actual = value.asInstanceOf[AnyRef].getClass

    if (isSubtypeOf(actual, ca.runtimeClass))
      fa(value.asInstanceOf[A])
    else if (isSubtypeOf(actual, cb.runtimeClass))
      fb(value.asInstanceOf[B])
    else
      throw new MatchError(
        s"Runtime class ${actual.getName} did not match either branch"
      )
  }
}

implicit def toFoldUnionOps[T](value: T): FoldUnionOps[T] =
  new FoldUnionOps(value)
```

---

## Using `foldUnion`

We can now define operations over union-like values.

### Example: `Int | String`

```scala
def size[T: (Int |∨| String)#λ](t: T): Int =
  t.foldUnion(
    (i: Int) => i,
    (s: String) => s.length
  )
```

Usage:

```scala
size(10)
// 10

size("hello")
// 5
```

Attempting to use an unsupported type fails at compile time:

```scala
size(3.14)
// does not compile
```

---

### Example: `Boolean | String`

```scala
def describe[T: (Boolean |∨| String)#λ](t: T): String =
  t.foldUnion(
    (b: Boolean) => if (b) "enabled" else "disabled",
    (s: String)  => s"message: $s"
  )
```

Usage:

```scala
describe(true)
// enabled

describe("system online")
// message: system online
```

---

## Limitations of the Scala 2 Approach

Although the encoding is elegant from a type-theoretical perspective, the implementation still depends on runtime class inspection and casting.

This introduces several limitations.

### Overlapping Types

Consider:

```scala
def example[T: (AnyRef |∨| String)#λ](t: T): String =
  t.foldUnion(
    (_: AnyRef) => "any reference",
    (_: String) => "string"
  )
```

Since `String` is also an `AnyRef`, the first branch will match first. Therefore, branch ordering becomes significant.

---

### Type Erasure

Generic types cannot be distinguished safely at runtime:

```scala
List[Int]
List[String]
```

Both erase to `List`, making runtime dispatch ambiguous.

---

### Null Values

The implementation explicitly rejects `null` values with a `MatchError` in order to avoid confusing runtime failures.

---

## Scala 3 and Native Union Types

Scala 3 introduces union types directly into the language:

```scala
def size(value: Int | String): Int =
  value match
    case i: Int    => i
    case s: String => s.length
```

This eliminates the need for type-level encodings entirely.

The resulting code is substantially simpler and more expressive.

---

## A Scala 3 `foldUnion`

Although direct pattern matching is generally preferable, it is still possible to define a `foldUnion` helper in Scala 3.

```scala
import scala.reflect.ClassTag

extension [A, B](value: A | B)
  def foldUnion[S](
    fa: A => S,
    fb: B => S
  )(using ca: ClassTag[A], cb: ClassTag[B]): S =
    value match
      case null =>
        throw MatchError("Cannot fold null as a union value")

      case v if ca.runtimeClass.isInstance(v) =>
        fa(v.asInstanceOf[A])

      case v if cb.runtimeClass.isInstance(v) =>
        fb(v.asInstanceOf[B])

      case v =>
        throw MatchError(
          s"Runtime class ${v.getClass.getName} did not match either branch"
        )
```

Usage:

```scala
def size(value: Int | String): Int =
  value.foldUnion(
    (i: Int) => i,
    (s: String) => s.length
  )
```

---

## Direct Pattern Matching in Scala 3

In most practical Scala 3 codebases, direct pattern matching is clearer and preferable:

```scala
def describe(value: Boolean | String): String =
  value match
    case b: Boolean => if b then "enabled" else "disabled"
    case s: String  => s"message: $s"
```

The union appears directly in the type signature:

```scala
Boolean | String
```

This removes the need for auxiliary encodings, type lambdas, and logical negation tricks.

---

## Comparison

### Scala 2

```scala
def size[T: (Int |∨| String)#λ](t: T): Int =
  t.foldUnion(
    (i: Int) => i,
    (s: String) => s.length
  )
```

### Scala 3

```scala
def size(t: Int | String): Int =
  t match
    case i: Int    => i
    case s: String => s.length
```

The Scala 2 version is primarily interesting as an exploration of advanced type-level programming techniques. The Scala 3 version, by contrast, expresses the same intent directly in the language.

---

## Conclusion

The Scala 2 `foldUnion` approach is a compelling example of how expressive Scala’s type system can be. Through logical encodings and type-level programming, developers were able to simulate union-like behavior years before native support existed.

However, despite the sophistication of the compile-time encoding, runtime dispatch still depends on type inspection and casting, which introduces practical limitations related to erasure and subtype overlap.

Scala 3 resolves these issues elegantly through native union types:

```scala
Int | String
```

As a result, code becomes simpler, clearer, and more maintainable while preserving the expressive power that earlier Scala 2 techniques attempted to approximate.
