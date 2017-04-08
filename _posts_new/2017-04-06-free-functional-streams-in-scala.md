---
layout: post
slug: free-functional-streams-in-scala
title: Free functional streams in Scala
description: This post investigates using free monads as the effect type in FS2 (functional streams for Scala) streams.
date: 2017-04-06
---

In this post we introduce the concept of free streams in Scala, where free is as in free monads. Streams are a data structure that represent a potentially infinite sequence of data, events, of effects. They are particularly well suited to modelling "online" processes, and are ideally suited to handle a wide variety of asynchronous and event driven data processing tasks. Much of the hardest and most error prone code, once framed in the context of a stream, becomes far easier to reason about and manage. The state-of-the-art [FS2 (functional streams for scala)](https://github.com/functional-streams-for-scala/fs2) in particular takes this even further by applying and integrating many important principles and practices of functional programming. In [previous posts]({% post_url 2016-12-08-free-monads-and-event-sourcing %}) we have looked at free monads and seen how elegant this abstraction can be, as applied to event sourcing. In this post we marry the two concepts, and get the best of both of them at once. We do this by creating a stream whose effect type is a free monad. If you don't know what that means, read on, we'll motivate for it and step-by-step derive the natural implementation. 

We'll start with a concrete use case, and slowly build it up. We are creating a trading system, and we writing the code to buy an asset such as a stock. Our use case is to place an order to buy or sell a stock at a certain price, and wait a certain time for that order to be filled. If the order is not filled in that time, the order must be cancelled. If the order is filled at any time, it returns immediately. All instructions are via a HTTP API, and it all has to happen asychronously and come back when it's done.

Firstly, lets start with the HTTP API. There are many HTTP APIs we could use, and most of them provide some asynchronous capability, typically in the form of returning a future which completes when the request returns with a response. So a single request would typically look something like this:

```scala
val responseFuture = Http.asyncPost(request)
val result = Await.result(responseFuture, 30.seconds)
```

`Future`, even though it's monadic, so we can `flatMap` the output from one request into the input of another, is still effectful. By this we mean the act of creating a future, in this case something like `Http.asyncGet(request)`, has an effect, in this case calling some external API.

Simply by wrapping this as a `Task`, we can defer this effect, and push it to the boundary of our program, which is where we want it to be. This is really low hanging fruit. If we use FS2's `Task` for example, we can wrap a future in a task with 

```scala
def fromFuture[A](fut: => Future[A])
```

where the future is only created when the task is executed.

So now we've got this out the way, we can assume our calls to the exchange API can all be considered as methods that return a `Task` rather than a future.

So here's our API:

```scala
def listTrades(apiKey: ApiKey, code: String): Task[Trades]

def getOrderBook(apiKey: ApiKey, pair: String): Task[OrderBook] 

def placeOrder(apiKey: ApiKey,
               code: String,
               orderType: String,
               volume: BigDecimal,
               price: BigDecimal): Task[OrderID] =

def stopOrder(apiKey: ApiKey, orderId: String): Task[Boolean]

def getOrder(apiKey: ApiKey, orderId: String): Task[Order]

def getBalances(apiKey: ApiKey): Task[List[Balance]]
```

From this we can define our simple order execution routine.

```scala
case class ExecutionDetails(trade: Option[Trade], balances: List[Balance])

def executeOrder(apiKey: ApiKey,
                 code: String,
                 orderType: String,
                 volume: BigDecimal,
                 price: BigDecimal,
                 verifyFrequency: FiniteDuration,
                 verifyAttempts: Long): Task[ExecutionDetails] = 
  for {
    orderId  <- placeOrder(apiKey, code, orderType, volume = volume, price = price)
    order    <- getOrderStatus(orderId, verifyFrequency, verifyAttempts)
    _        <- if (order.state == "COMPLETE") Task.now(false) else stopOrder(apiKey, orderId.id)
    trade    <- getTrade(apiKey, order)
    balances <- getBalances(apiKey)
  } yield ExecutionDetails(trade, balances)
```

Our execution method returns an `ExecutionDetails` structure, with an optional `Trade` object that's defined if the order was filled, and the new account balances. 

If you look closely at the above implementation you'll see that we haven't defined the `getOrderStatus` method. This is where FS2 comes in. What we do is we poll the API every so often, until we finally give up. If we give up, we then cancel the order before finishing up and returning. 

This is a simple task for FS2. The following code will get the job done. 

```scala
def getOrderStatus(orderId: String, verifyFrequency: FiniteDuration, verifyAttempts: Long): Task[Order] =
  fs2.Stream
    .eval[Task, Order](Task.schedule(getOrder(apiKey, orderId)), verifyFrequency)
    .repeat
    .takeThrough(_.state != "COMPLETE")
    .take(verifyAttempts)
    .runLast
    .map(_.get) 
```

If you know FS2, this code is dead straightforward. If you're not familiar with FS2, the steps are as follows:

1. `eval[Task, Order](Task.schedule(getOrder(apiKey, orderId)), verifyFrequency)` execute a task afer delay of `verifyFrequency`. The result, of type `Order`, is emitted to the stream. This is where the work gets done.
2. `repeat` tells it to do exactly that, indefinitely.
3. `takeThrough(_.state == "PENDING")` tells the stream to continue emitting elements as long as `_.state == "PENDING"`, but also emit the first element in which this condition fails.
4. `take(verifyAttempts)` tells the stream to emit at most `verifyAttempts` elements.
5. `runLast` "runs" the stream and returns an optional the last element, which we then `get`, as we know the stream is non empty.

Note that we've put "runs" in quotes, as it doesn't really run the stream, it just transforms the stream into a `Task` that returns a vector when the task is run. Running the stream is a two stage process. In this case the second stage will only get executed when we run the final task. For example:

```scala
val executeTask: Task[ExecutionDetails] = executeOrder(apiKey, "AAPL", "BID", 100.0, 125.0, 10.seconds, 30)
val executeAttempt: Either[Throwable, ExecutionDetails] = executeTask.unsafeAttemptRun
```

So this is very simple code - far simpler than any handcrafted imperative equivalent, or some kind of recursive scheme.

So now we have our order execution code, and now we want to run it. But hang on a second, we haven't tested it, and we can't afford it to fail - if things go wrong, we could lose money. The traditional way to test this code would be to mock the API. That's doable, but free monads provide a cleaner abstraction that eliminate the need for mocking frameworks and any of their underlying magic. Wouldn't it be nice if we could do that here? And it turns out we can.

Just a brief recap on free monads: Firstly a monad is a data structure that best represents units of computation that can be composed sequentially - the of one forms the input of the next. Free monads separate the *what* gets computed, from the *how*. A free monad stack consists of:
* An *algebra*, which is a specification of the computations that can occur, defined in Scala as a sealed trait family of case classes. This is the *what*.
* A special monad `Free[F[_], A]` (which is a monad over the second paramter `A` and where the parameter `F` is the algebra), and a `liftF` function that "lifts" the algebra into the monad.
* An *interpreter*, which is a natural transformation from `F` to a target implementation monad `G`, usually written `F ~> G`. The target monad is often `Task`, but can be any other monad, depending on the use case. This is the *how*.

Let's start by abstracting our API as a free monad. We start by defining our algebra of operations, an `OrderOp` sealed trait family of case classes, and an `OrderFree` free monad.

```scala
trait FreeOp[F[_], A] { this: F[A] =>
  def liftF: Free[F, A] = Free.liftF(this)
}

sealed trait OrderOp[A] extends FreeOp[OrderOp, A]
type OrderFree[A] = Free[OrderOp, A]

case class ListTrades(code: String) extends OrderOp[Trades]

case class GetOrderBook(pair: String) extends OrderOp[OrderBook] 

case class PlaceOrder(code: String,
               orderType: String,
               volume: BigDecimal,
               price: BigDecimal) extends OrderOp[OrderID] =

case class StopOrder(orderId: String) extends OrderOp[Boolean]

case class GetOrder(orderId: String) extends OrderOp[Order]

case class GetBalances() extends OrderOp[List[Balance]]
```

Note that we've remove the `apiKey` parameter, as we can consider this an implementation detail. The `FreeOp` base trait is a convenient mechanism for providing the `liftF` 

Now that we've done this we can rewrite our order execution program.

```scala
case class ExecutionDetails(trade: Option[Trade], balances: List[Balance])

def executeOrder(apiKey: ApiKey,
                 code: String,
                 orderType: String,
                 volume: BigDecimal,
                 price: BigDecimal,
                 verifyFrequency: FiniteDuration,
                 verifyAttempts: Long): OrderFree[ExecutionDetails] = 
  for {
    orderId  <- PlaceOrder(code, orderType, volume = volume, price = price).liftF
    order    <- getOrderStatus(orderId, verifyFrequency, verifyAttempts)
    _        <- if (order.state == "COMPLETE") Free.pure[OrderOp, Boolean](false) else StopOrder(orderId.id).liftF
    trade    <- GetTrade(order).liftF
    balances <- GetBalances()
  } yield ExecutionDetails(trade, balances)
```

Noticed how it's hardly changed at all!

All we need to do is modify the FS2 part. But before we do this lets take a look at the `Stream` type in FS2. Stream is a class of type `Stream[F[_],O]`. `O` is the output type and `F`, referred to as the *effect* type, is a monad. `Task` is often chosen as the effect type, often out of convenience, but in some ways it is a poor choice. It is the most general effect type that can represent any abitrary effect. It is the `Any` of effect types, as it imposes no constraint on what effect can occur.

Our objective is to modify `getOrderStatus` to return a `OrderFree[Order]` instead of a `Task[Order]`. Working backwards, it seems clear that what we need is to generate a stream of type `Stream[OrderFree,Order]`, and when we run this with `runLast`, it generates returns a `OrderFree[Order]`. We don't need to change the code much to make this happen.

```scala
def getOrderStatus(orderId: String): OrderFree[Order] =
  fs2.Stream
    .eval[OrderFree, Order](Delay(verifyFrequency).liftF.flatMap(_ => GetOrder(orderId).liftF))
    .repeat
    .takeThrough(_.state == "PENDING")
    .take(verifyAttempts)
    .runLast
    .map(_.get) 
```

As part of this refactoring we've made a change to our free algebra by adding the following operation:

```scala
final case class Delay(duration: FiniteDuration) extends OrderOp[Unit]
```

This is based on the recognition that having a delay at this stage is a domain concern rather than an implementation detail. Then `Delay(verifyFrequency).liftF.flatMap(_ => GetOrder(orderId).liftF)` indicates on operation that get's the order status after a delay of `verifyFrequency`.

This all looks good in the IDE, but when we try to compile the compiler complains:

```scala
[error] could not find implicit value for parameter F: fs2.util.Catchable[OrderFree]
[error]         .runLast
[error]          ^
```

Oh drat, should have known this might happen! One of the features of FS2 is exception safety. A FS2 `Stream` object will never throw an exception. The price of this is that it the effect monad has to do it's share by carrying the exception through the computation. And the purpose of the `Catchable` typeclass is to tell FS2 how it's going to do this. So we need to provide an implicit instance of `Catchable[OrderFree]`.

Here is the `Catchable` definition:

```scala
trait Catchable[F[_]] extends Monad[F] {
  /** Lifts a pure exception in to the error mode of the `F` effect. */
  def fail[A](err: Throwable): F[A]
  /** Provides access to the error in `fa`, if present, by wrapping it in a `Left`. */
  def attempt[A](fa: F[A]): F[Attempt[A]]
}
```

So reading the comments, it looks like our free monad has to have an error mode. What can we do? The first thing that came to my mind, is what if we ignore this and we implement `fail` by just throwing? But immediately I thought better of this, as that's surely going to break FS2 exception safety. The next thought is what if we wrap our free monad in another monad with an error mode, like `Attempt`, which is just an alias for an `Either[Throwable,?]`. That sounds doable, but it's surely going to result in a lot of extra boiler plate code that's going to get in the way. It turns out there is a simpler solution. We augment our free algebra with an error operation:

```scala
final case class OrderError[A](throwable: Throwable) extends OrderOp[A]
```

Having this, our `Catchable` implementation is quite straightforward:

```scala
implicit def catchableOrder(implicit M: Monad[OrderFree]) = new Catchable[OrderFree] {
  override def fail[A](err: Throwable) = OrderError(err).liftF
  override def attempt[A](fa: OrderFree[A]) = fa.map {
    case OrderError(throwable) => Left(throwable)
    case a                     => Right(a)
  }
  override def flatMap[A, B](a: LunoFree[A])(f: (A) => LunoFree[B]): LunoFree[B] = M.flatMap(a)(f)
  override def pure[A](a: A): LunoFree[A]                                        = M.pure(a)
}
```

This works comiples and perfectly. When FS2 encounters an exception, it calls `fail` on the `Catchable[OrderFree]` typeclass instance, which stuffs the error into the `OrderError` object. 

Because `Catchable` extends `Monad` it also needs to provide an interpretation of `pure` and `flatMap`. I'm not sure why it needed to be a `Monad`, this seems to add unnecessary noise.  

The next step is to create an interpreter. This is very straightforward - here is our interpreter to `Task`:

```scala
def orderOp2Task(apiKey: ApiKey)(implicit catchable: Catchable[Task]) = new (OrderOp ~> Task) {
  override def apply[A](fa: OrderOp[A]): Task[A] = fa match {
    case ListOrders(pair)                           => listOrders(apiKey, code)
    case ListTrades(pair)                           => listTrades(apiKey, code)
    case GetOrderBook(pair)                         => getOrderBook(apiKey, code)
    case StopOrder(orderId)                         => stopOrder(apiKey, orderId)
    case GetOrder(orderId)                          => getOrder(apiKey, orderId)
    case GetBalances()                              => getBalances(apiKey)
    case GetTrade(order)                            => getTrade(apiKey, order)
    case PlaceOrder(pair, orderType, volume, price) => placeOrder(apiKey, code, orderType, volume, price)
    case Delay(duration)                            => Task.schedule((), duration)
    case OrderError(throwable)                      => catchable.fail(throwable)
  }
}
```

Most of the methods simply delegate to the corresponding API methods. The only exceptions are the additional operations we've added. `Delay` has to have a bespoke interpretation, which here is a `Task` that just returns unit after a delay of `duration`. The `OrderError` interpretation just has to pass the error state into the error mode of the target interpreter. For this reason, an `implicit catchable: Catchable[Task]` parameter is required. We pass the API key as a parameter into the interpreter.

Finally, to run the new free program, a simple extra interpretation step is required to turn our free monad into a runnable task:

```scala
val executeFree = executeOrder("AAPL", "BID", 100.0, 125.0, 10.seconds, 30)
val executeTask: Task[ExecutionDetails] = executeFree.foldMap(orderOp2Task(apiKey))
val executeAttempt = executeTask.unsafeAttemptRun
```

So, as always, our code gets more descriptive power if we use a more constrained type, and exactly the same applies here.

Then we create an interpreter:
```scala
```
