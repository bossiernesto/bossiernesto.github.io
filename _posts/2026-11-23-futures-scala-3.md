---
layout: posts
title: "Futures en Scala 3: composición, errores y evaluación diferida"
excerpt: "Cómo crear y componer Futures, manejar fallos y distinguirlos de Promise y de una computación lazy."
categories:
  - Scala
  - Functional Programming
tags:
  - Scala 3
  - Concurrency
  - Futures
  - Promises
---

# Futures en Scala 3: trabajo asíncrono y valores que aún no llegaron

Cuando una operación puede tardar —consultar un servicio, leer datos o calcular algo costoso— no siempre conviene detener el flujo que la inició hasta tener el resultado. En Scala, `Future[A]` permite representar el resultado eventual de una operación cuyo tipo de éxito es `A`.

El nombre puede inducir a error: un `Future` no es, por sí mismo, una receta que espera a que alguien la ejecute. Tampoco es el objeto que normalmente completa el resultado. Para entenderlo bien hay que separar tres ideas:

- **`Future[A]`:** una referencia de solo lectura al resultado eventual, exitoso o fallido.
- **`Promise[A]`:** el objeto que permite completar explícitamente ese resultado.
- **Una computación diferida:** una receta que todavía no comenzó y que puede iniciarse más adelante.

En este artículo vamos a ver cómo se crean y componen Futures, qué papel cumple `ExecutionContext` y por qué `Future` no equivale ni a `Promise` ni a un valor lazy.

## El `ExecutionContext`

Las operaciones asíncronas necesitan un `ExecutionContext`: el mecanismo que ejecuta los bloques y callbacks asociados a un `Future`. En Scala 3 se puede proporcionar como un valor `given`:

```scala
import scala.concurrent.ExecutionContext

given ExecutionContext = ExecutionContext.global
```

También se puede pedir explícitamente en las funciones de una biblioteca:

```scala
import scala.concurrent.{ExecutionContext, Future}

def calculate()(using ec: ExecutionContext): Future[Int] =
  Future {
    6 * 7
  }
```

`ExecutionContext.global` es práctico para ejemplos y para trabajo breve que no bloquea. En una aplicación real, la elección debe considerar qué trabajo se ejecuta: el pool global no es una buena casa para operaciones bloqueantes y largas, porque pueden ocupar sus hilos e impedir que otras tareas avancen. Para I/O, suele ser preferible un cliente realmente asíncrono; si se debe llamar a una API bloqueante, conviene aislarla en un executor dedicado. `blocking { ... }` puede informar al contexto de ciertos bloqueos, pero no vuelve no bloqueante a esa API.

Un `ExecutionContext` no promete que cada callback corra en un hilo nuevo ni que dos tareas vayan a ejecutarse en paralelo. Decide cómo ejecutar el trabajo; el paralelismo efectivo depende también de los recursos disponibles y de esa implementación.

## Crear un `Future`

La forma habitual de programar una computación es `Future { ... }`:

```scala
import scala.concurrent.{ExecutionContext, Future}

def calculate(): Int =
  (1 to 1000).sum

def calculateAsync()(using ec: ExecutionContext): Future[Int] =
  Future {
    calculate()
  }
```

Al evaluar `Future { ... }`, el bloque se entrega al `ExecutionContext` para ejecutarse y se devuelve un `Future[Int]`. El llamador puede continuar sin esperar el valor. Cuando el cálculo termina, el Future se completa con el resultado; si el bloque lanza una excepción no fatal, se completa con un fallo.

También existen constructores para resultados que ya se conocen:

```scala
val cached: Future[Int] = Future.successful(42)
val failed: Future[Int] =
  Future.failed(new IllegalStateException("no disponible"))
```

`Future.successful` no programa un cálculo: crea un Future ya completado. El argumento se evalúa antes de llamar al método, porque Scala evalúa los argumentos de forma estricta.

## Componer operaciones

La idea central es transformar y encadenar Futures sin bloquear:

- `map` transforma el valor de éxito, por ejemplo `Future[Int]` en `Future[String]`.
- `flatMap` conecta una operación que ya devuelve otro Future.
- Si un Future falla, los siguientes pasos de `map` y `flatMap` no procesan un valor inexistente; el fallo continúa por la cadena hasta que se maneje.

Por ejemplo, si buscar una cuenta devuelve un Future y luego se necesitan sus movimientos:

```scala
import scala.concurrent.{ExecutionContext, Future}

final case class Account(id: Long)
final case class Transaction(id: Long, amount: BigDecimal)
final case class Statement(account: Account, transactions: List[Transaction])

trait BankingApi:
  def account(id: Long): Future[Account]
  def transactions(accountId: Long): Future[List[Transaction]]

def statement(
    api: BankingApi,
    accountId: Long
)(using ec: ExecutionContext): Future[Statement] =
  for
    account <- api.account(accountId)
    transactions <- api.transactions(account.id)
  yield Statement(account, transactions)
```

La consulta de movimientos depende del resultado de `account`, así que empieza cuando la primera operación termina correctamente. Una *for-comprehension* sobre Futures es azúcar sintáctica para encadenar `flatMap` y `map`; no transforma una secuencia dependiente en trabajo paralelo.

En cambio, dos consultas independientes se pueden iniciar antes de combinarlas:

```scala
def dashboard(
    api: BankingApi,
    accountId: Long
)(using ec: ExecutionContext): Future[(Account, List[Transaction])] =
  val accountF = api.account(accountId)
  val transactionsF = api.transactions(accountId)

  for
    account <- accountF
    transactions <- transactionsF
  yield (account, transactions)
```

Las llamadas a `api` se realizan antes de entrar en la comprensión; si ambas devuelven operaciones asíncronas, pueden avanzar al mismo tiempo. El `for` combina los resultados. El orden del código, por sí solo, no determina si dos operaciones independientes se ejecutan en paralelo: eso depende de cuándo se inician y del `ExecutionContext`.

Para una colección, `Future.traverse` aplica una función asíncrona a sus elementos y reúne los resultados:

```scala
def loadAll(
    ids: List[Long],
    api: BankingApi
)(using ec: ExecutionContext): Future[List[Account]] =
  Future.traverse(ids)(api.account)
```

Los resultados quedan en el mismo orden que los identificadores. Si `ids` es muy grande, este ejemplo puede iniciar muchas operaciones a la vez: `Future.traverse` no limita la concurrencia. Para cargas grandes hace falta aplicar contrapresión o establecer un límite con una herramienta apropiada.

## Manejar errores y observar el resultado

Un Future termina en uno de dos estados: éxito con un valor `A` o fallo con una excepción. Para recuperar un error conocido se puede usar `recover`; para reemplazarlo por otra operación asíncrona, `recoverWith`:

```scala
final case class AccountNotFound(id: Long)
    extends RuntimeException(s"Account $id not found")

val accountF: Future[Account] =
  api.account(accountId).recover {
    case _: AccountNotFound => Account(accountId)
  }
```

El tipo sigue siendo `Future[Account]`: el caso recuperado se convierte en un éxito. Conviene recuperar errores que realmente se pueden manejar, en vez de ocultar cualquier excepción con un valor genérico.

`onComplete` permite observar el resultado como `Try[A]`, incluyendo ambos caminos:

```scala
import scala.util.{Failure, Success}

accountF.onComplete {
  case Success(account) => println(s"Cuenta: $account")
  case Failure(error)   => println(s"Falló: ${error.getMessage}")
}
```

Es útil en límites donde hace falta una acción final —por ejemplo, registrar un resultado—, pero no suele ser la mejor forma de construir una cadena: el valor que retorna el callback se descarta. Para transformar resultados, preferí `map`, `flatMap`, `recover` y `recoverWith`. El orden de varios callbacks registrados con `onComplete` no está garantizado.

`Await.result` permite esperar bloqueando por el resultado. Puede servir en pruebas o en una frontera de compatibilidad, pero usarlo dentro de otra operación asíncrona ocupa un hilo y puede agotar el pool. En general, es mejor devolver el Future y componerlo hasta el punto que realmente necesita el resultado.

## Qué es y qué no es un `Promise`

`Future` y `Promise` están relacionados, pero cumplen roles distintos. Un Promise crea un resultado que alguien completará; su método `future` entrega la vista de solo lectura que otros pueden componer:

```scala
import scala.concurrent.{Future, Promise}

val promise = Promise[Int]()
val result: Future[Int] = promise.future

promise.trySuccess(42)
```

Un Promise solo puede completarse una vez, con éxito o con fallo. `trySuccess`, `tryFailure` y `tryComplete` devuelven `false` si ya estaba completado, por lo que son útiles cuando puede haber carreras entre varias rutas de finalización. Las variantes `success` y `failure` lanzan una excepción si se intenta completarlo de nuevo.

En el flujo habitual, el consumidor recibe `Future[A]`, no `Promise[A]`: no debería decidir cuándo ni con qué valor termina el trabajo de otra parte del programa. La API que inicia una operación suele devolver el Future; internamente, un Promise sirve para adaptarla cuando el resultado llega desde afuera, por ejemplo mediante un callback:

```scala
import scala.concurrent.{Future, Promise}
import scala.util.Try

def fromCallback[A](
    register: (Try[A] => Unit) => Unit
): Future[A] =
  val promise = Promise[A]()
  register { result =>
    promise.tryComplete(result)
    ()
  }
  promise.future
```

La función `register` representa una API de callbacks que más tarde informa `Success(value)` o `Failure(error)`. El adaptador devuelve un Future inmediatamente; cuando llega el callback, completa el Promise y los consumidores observan el resultado. Si la API pudiera llamar al callback más de una vez, `tryComplete` evita que la segunda finalización vuelva a completar la promesa.

Por eso un Promise no es “un Future que todavía no empezó”: es el mecanismo mutable de una sola escritura para completar el Future asociado.

## Por qué un `Future` no es una computación diferida

Una computación diferida es una receta pendiente. Un `Future[A]`, en cambio, describe el resultado eventual de un trabajo. En particular, `Future { ... }` entrega el bloque al `ExecutionContext` al crearse; no espera a que alguien lea el resultado:

```scala
def calculateLater(): Future[Int] =
  Future {
    calculate()
  }
```

Cada llamada a `calculateLater()` crea y programa un Future nuevo. Si nadie consulta el resultado, eso no cancela ni deshace el trabajo. El Future puede haberse completado ya, estar ejecutándose, o —si se construyó desde un Promise— seguir esperando que algún productor lo complete.

Comparalo con una función diferida y con un `lazy val`:

```scala
val thunk: () => Int =
  () => calculate()             // no calcula hasta invocar thunk()

lazy val cached: Int =
  calculate()                   // calcula al primer acceso y guarda el resultado

val running: Future[Int] =
  Future { calculate() }        // programa el trabajo ahora
```

La función `thunk` puede ejecutarse cada vez que se invoca. El `lazy val` demora la evaluación y conserva el valor obtenido. El Future normalmente programa la operación cuando se construye, y conserva su resultado eventual: no ofrece por sí mismo ni evaluación bajo demanda, ni memoización de la receta, ni cancelación. También puede representar un valor ya calculado, como muestra `Future.successful`.

Si necesitás posponer incluso el inicio del Future, podés devolver una función:

```scala
val startWhenAsked: () => Future[Int] =
  () => Future { calculate() }
```

Cada invocación inicia una operación distinta. Para compartir una sola operación, se puede compartir el Future creado; para describir efectos que se ejecutarán bajo el control explícito de un runtime, bibliotecas como Cats Effect (`IO`) o ZIO (`Task`) ofrecen modelos distintos. No son Futures de la biblioteca estándar: tienen semánticas y herramientas propias para composición, cancelación y recursos.

## Límites importantes

- **No hay cancelación incorporada.** Dejar de conservar una referencia al Future no cancela la tarea. Si la operación necesita cancelación, el cliente o el modelo de efectos debe ofrecerla.
- **Asincronía no significa paralelismo garantizado.** Un `ExecutionContext` puede serializar tareas, ejecutarlas en el hilo actual o repartirlas entre varios hilos.
- **No bloquees el pool sin necesidad.** Si una llamada síncrona puede esperar mucho tiempo, aislarla en un executor apropiado o usar una API no bloqueante.
- **Un Future no se reinicia.** Si necesitás repetir una operación, llamá a una función que cree un nuevo Future y definí explícitamente la política de reintentos.
- **Las carreras son parte del modelo.** El momento de terminación y el orden de callbacks no deben usarse como mecanismo de sincronización implícito.

## En resumen

`Future[A]` es una representación composable de un resultado que puede llegar más tarde, con éxito o con error. `map` transforma el resultado, `flatMap` permite encadenar dependencias y `ExecutionContext` ejecuta los bloques y callbacks. Para observar un resultado no hace falta bloquear; para manejar fallos existen `recover` y `recoverWith`.

`Promise[A]` es el productor de una sola escritura que puede completar el Future asociado. Una función o un `lazy val` pueden representar trabajo diferido; `Future { ... }` normalmente programa el trabajo al crearse. Distinguir estos roles evita iniciar tareas antes de tiempo, saturar el pool o esperar de forma bloqueante cuando la composición asíncrona alcanza.

## Referencias

- Scala 3 API, [`Future`](https://www.scala-lang.org/api/3.x/scala/concurrent/Future.html).
- Scala 3 API, [`Promise`](https://www.scala-lang.org/api/3.x/scala/concurrent/Promise.html).
- Scala Documentation, [Futures and Promises](https://docs.scala-lang.org/overviews/core/futures.html).
