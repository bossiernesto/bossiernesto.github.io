---
layout: posts
title: "Intro to Monad Writers and Readers in Scala"
excerpt: "Monad Writers and Readers. Use in Scala"
image:
  path: /assets/blog-images/monad.png
categories:
  - Fucntional Programming
  - Scala
tags:
  - Scala
  - Monads
  - Functional Programaming
---

# Understanding Reader and Writer Monads in Scala

## Intro 

In functional programming, monads are a design pattern used to structure computations in a way that keeps side effects, context, or sequencing under control while still allowing code to remain composable and declarative. A monad can be understood as a type constructor together with two essential operations: one that places a value into the monadic context (commonly called pure or unit) and another that chains computations while preserving that context (commonly called flatMap or bind). In Scala, monads are deeply integrated into the language through constructs such as Option, Future, Either, and collections. The for comprehension syntax is syntactic sugar over repeated calls to flatMap and map, which makes monadic code readable while still preserving the mathematical structure underneath. Monads help developers model computations that may fail, computations that are asynchronous, computations with shared environments, or computations that accumulate effects, all without scattering imperative state management throughout the codebase.

### Intro to Reader Monad

The Reader monad is a specialized monad used to model computations that depend on a shared, immutable environment. Instead of passing configuration or contextual objects manually through every function call, the Reader monad encapsulates functions of the form Env => A, where Env is the shared environment and A is the resulting value. Conceptually, the Reader monad represents “a computation that can read from some context.” This is particularly useful in applications where many services depend on common configuration, database connections, repositories, or runtime context. In Scala, the Reader monad is commonly implemented either manually or through libraries such as Cats. Using Readers promotes dependency injection through composition rather than mutable global state or heavyweight frameworks. A Reader computation can be combined with others while implicitly threading the environment through all steps. For example, if an application has a configuration object containing database settings and API keys, multiple Reader computations can access this configuration without explicitly receiving it as a parameter in every intermediate function.

In Scala, a simple Reader monad can be represented as a case class wrapping a function from an environment to a value. The map operation transforms the produced value while leaving the environment untouched, and flatMap allows chaining computations that themselves depend on the same environment. This enables elegant composition using for comprehensions. A typical Reader implementation might define case class Reader[E, A](run: E => A). The flatMap method takes a function from A to another Reader[E, B] and produces a new Reader that feeds the same environment through both computations. This allows business logic to remain pure and testable because dependencies are represented explicitly as values rather than hidden global state. In large Scala systems, Reader-based patterns are often used alongside tagless-final architectures or effect systems such as Cats Effect and ZIO.

### Intro to Writer Monad

The Writer monad, in contrast, is designed to carry along an additional accumulated output while computations execute. Rather than reading from a shared environment, the Writer monad augments computations with a log, trace, or auxiliary piece of information. Conceptually, a Writer represents a pair (Log, A), where A is the computation result and Log is some accumulated metadata. The log type must support combination, usually through a mathematical structure called a monoid, meaning there is an identity value and an associative way to combine logs. In practice, Writer monads are useful for debugging, audit trails, execution traces, metrics collection, or compiler passes where intermediate information should be preserved without introducing mutable variables or side effects.

In Scala, the Writer monad can also be implemented manually or obtained through functional libraries like Cats. A basic implementation might use case class Writer[L, A](log: L, value: A). The map operation transforms the value while preserving the log, whereas flatMap combines the logs from sequential computations while propagating the resulting value forward. Because log accumulation is automatic, developers can build pipelines of computations that transparently collect information about what occurred during execution. For example, a sequence of arithmetic operations could record every intermediate step without manually appending strings throughout the code. This becomes especially powerful when the log is not merely text but structured data such as vectors of events or performance statistics.

Reader and Writer monads illustrate two important aspects of functional abstraction in Scala. The Reader monad models dependency propagation, while the Writer monad models effect accumulation. Together, they demonstrate how monads generalize computation patterns into reusable compositional structures. In real-world Scala development, these monads are often combined with others through monad transformers or effect systems, enabling applications to manage configuration, logging, errors, asynchronous execution, and state in a principled and type-safe manner. Rather than relying on implicit mutable state or framework magic, Scala’s monadic approach emphasizes explicit modeling of computational behavior through types, leading to code that is easier to reason about, test, and compose.

### Quick and practical definition of Monads

In functional programming abstractions that initially seem complex, but solve very practical problems once understood. One of the most important abstractions is the **Monad**. In Scala, monads are everywhere: `Option`, `Future`, `Either`, collections, and many functional libraries all rely on the same compositional ideas.

A monad can be understood as a structure that wraps values inside a context while providing a way to chain computations together. In Scala, this chaining usually happens through `map` and `flatMap`, which also power the language’s `for` comprehensions.

Two particularly interesting monads are the **Reader** and **Writer** monads. While they are less commonly discussed than `Option` or `Future`, they demonstrate how functional programming models dependency management and effect accumulation in a purely compositional way.

---

## The Reader Monad

The **Reader monad** models computations that depend on a shared environment.

Instead of manually passing configuration or services through every function call, Reader encapsulates computations of the form:

```scala
Env => A
```

Where:

- `Env` is some shared environment or dependency
- `A` is the resulting value

This is especially useful for dependency injection, configuration management, repositories, or service composition.

## A Simple Reader Implementation

```scala
case class AppConfig(apiUrl: String, apiKey: String)

case class Reader[Env, A](run: Env => A) {

  def map[B](f: A => B): Reader[Env, B] =
    Reader(env => f(run(env)))

  def flatMap[B](f: A => Reader[Env, B]): Reader[Env, B] =
    Reader(env => f(run(env)).run(env))
}
```

Here, `Reader` wraps a function that requires an environment.

Now we can define computations that depend on `AppConfig`:

```scala
def getApiUrl: Reader[AppConfig, String] =
  Reader(config => config.apiUrl)

def getApiKey: Reader[AppConfig, String] =
  Reader(config => config.apiKey)
```

And compose them:

```scala
val program: Reader[AppConfig, String] =
  for {
    url <- getApiUrl
    key <- getApiKey
  } yield s"Calling $url with key $key"
```

Finally:

```scala
val config = AppConfig(
  apiUrl = "https://api.example.com",
  apiKey = "secret-key"
)

println(program.run(config))
```

Output:

```scala
Calling https://api.example.com with key secret-key
```

### Why is this useful?

Without Reader, every function would need to explicitly receive `config` and manually pass it forward.

Reader allows the environment to flow implicitly through the computation while still remaining purely functional and testable.

This becomes very powerful in larger systems where many services depend on shared runtime configuration.

---

## The Writer Monad

While Reader propagates a shared environment, the **Writer monad** accumulates additional information alongside computations.

A Writer usually represents:

```scala
(Log, A)
```

Where:

- `Log` is accumulated metadata
- `A` is the resulting value

This is useful for:

- Logging
- Tracing execution
- Audit systems
- Compiler passes
- Metrics collection

without relying on mutable state.

---

### A Simple Writer Implementation

```scala
case class Writer[Log, A](log: Log, value: A) {

  def map[B](f: A => B): Writer[Log, B] =
    Writer(log, f(value))

  def flatMap[B](f: A => Writer[Log, B])
                (implicit monoid: Monoid[Log]): Writer[Log, B] = {

    val next = f(value)

    Writer(
      monoid.combine(log, next.log),
      next.value
    )
  }
}
```

The Writer requires a way to combine logs, so we define a `Monoid`.

```scala
trait Monoid[A] {
  def empty: A
  def combine(x: A, y: A): A
}
```

Example for logs stored as `List[String]`:

```scala
implicit val stringListMonoid: Monoid[List[String]] =
  new Monoid[List[String]] {

    def empty: List[String] = Nil

    def combine(
      x: List[String],
      y: List[String]
    ): List[String] =
      x ++ y
  }
```

Now we can define computations that accumulate logs.

```scala
def addOne(x: Int): Writer[List[String], Int] =
  Writer(
    List(s"Added one to $x"),
    x + 1
  )

def multiplyByTwo(x: Int): Writer[List[String], Int] =
  Writer(
    List(s"Multiplied $x by two"),
    x * 2
  )
```

And compose them:

```scala
val result =
  for {
    a <- addOne(3)
    b <- multiplyByTwo(a)
  } yield b
```

Reading the result:

```scala
println(result.value)
println(result.log)
```

Output:

```scala
8

List(
  Added one to 3,
  Multiplied 4 by two
)
```

### Why is this useful?

Instead of mutating a global logger or printing directly inside functions, Writer allows logs to be accumulated as part of the computation itself.

This keeps the code pure and composable while still preserving execution information.

---

## Using Reader and Writer with Cats

In real Scala projects, developers usually rely on functional libraries like Cats instead of implementing monads manually.

### Reader with Cats

```scala
import cats.data.Reader
import cats.implicits._

case class Config(databaseUrl: String)

val readDatabaseUrl: Reader[Config, String] =
  Reader(config => config.databaseUrl)

val readerProgram: Reader[Config, String] =
  for {
    db <- readDatabaseUrl
  } yield s"Connecting to $db"

println(
  readerProgram.run(
    Config("localhost:5432")
  )
)
```

### Writer with Cats

```scala
import cats.data.Writer
import cats.implicits._

type Logged[A] = Writer[List[String], A]

def divide(x: Int, y: Int): Logged[Int] =
  Writer(
    List(s"Dividing $x by $y"),
    x / y
  )

def increment(x: Int): Logged[Int] =
  Writer(
    List(s"Incrementing $x"),
    x + 1
  )

val loggedProgram =
  for {
    a <- divide(10, 2)
    b <- increment(a)
  } yield b

val (logs, value) =
  loggedProgram.run

println(value)
println(logs)
```

---

# Final Thoughts

The Reader and Writer monads showcase how functional programming can model complex application behavior in a purely compositional way. The **Reader** propagates shared dependencies and configuration, while the **Writer** accumulates logs and metadata.
Both help avoid hidden mutable state while improving modularity, composability, and testability. In Scala, these abstractions become even more powerful when combined with libraries such as Cats, Cats Effect, or ZIO, forming the foundation for modern functional application design.