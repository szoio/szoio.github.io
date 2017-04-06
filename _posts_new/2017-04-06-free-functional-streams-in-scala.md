---
layout: post
slug: free-functional-streams-in-scala
title: Free functional streams in Scala
description: This post investigates using free monads as the effect type in FS2 (functional streams for Scala) streams.
date: 2017-04-06
---

In this post we introduce the concept of free streams in Scala. In previous posts we have already looked at free monads and seen how elegant this abstraction can be, as applied to event sourcing. Streams are a data structure that represent a potentially infinite sequence of data, events, of effects. They are particularly well suited to modelling "online" processes, and are ideally suited to handle a wide variety of asynchronous and event driven data process tasks. Much of the hardest and most error prone code, once framed in the context of a stream, becomes far easier to reason about and manage. The state-of-the-art [FS2 (functional streams for scala)](https://github.com/functional-streams-for-scala/fs2) in particular takes this even further by applying and integrating many important principles and practices of functional programming. In this post we marry the two concepts, and get the best of both of them at once, by creating a stream whose effect type is a free monad. If you don't know what that means, read on, we'll motivate for it and step-by-step derive the natural implementation. 

We'll start with a concrete use case, and slowly build it up. We are creating a trading system, and we writing the code to buy an asset such as a stock. Our use case is to place an order to buy or sell a stock at a certain price, and wait a certain time for that order to be filled. If the order is not filled in that time, the order must be cancelled. If the order is filled at any time, it returns immediately. All instructions are via a HTTP API, and it all has to happen asychronously and come back when it's done.

Firstly, lets start with the HTTP API. There are many HTTP APIs we could use, and most of them provide some asynchronous capability, typically in the form of returning a future which completes when the request returns with a response. So a single request would typically look something like this:

```scala
val responseFuture = Http.asyncPost(request)
val result = Await.result(responseFuture, 30.seconds)
```

If we want to result of one request to be fed into the request of another, we'll typically `flatMap` it, so it may look like this.

```scala
val responseFuture2 = Http.asyncGet(request1).flatMap { result1 =>
  val request2 = someLogic(result1)
  Http.asyncPost(request2)
}
val result2 = Await.result(responseFuture2, 30.seconds)
```

Obviously if there's more than a couple of chained requests, we'd map them in a for comprehension.

`Future`, even though it's monadic, is still effectful. By this we mean the act of creating a future, in this case something like `Http.asyncGet(request)`, has an effect, in this case calling some external API.

Simply wrapping this as a `Task`, we can defer this effect, and push it to the boundary of our program, which is where we want it to be. This is really low hanging fruit. If we use FS2's `Task` for example, we can wrap a future in a task with 

```scala
def fromFuture[A](fut: => Future[A])
```

where the future is only created when the task is executed.

So now we've got this out the way, we can assume our calls to the exchange API can all be considered as methods that return a `Task` rather than a future.

So here's our API.

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

 def attemptExecute(apiKey: ApiKey,
                     code: String,
                     orderType: String,
                     volume: BigDecimal,
                     price: BigDecimal,
                     verifyFrequency: FiniteDuration,
                     verifyAttempts: Long): Task[ExecutionDetails] = 
    for {
      orderId  <- placeOrder(apiKey, code, orderType, volume = volume, price = price)
      order    <- getOrderStatus(orderId, verifyFrequency, verifyAttempts)
      _        <- if (order.state == "PENDING") stopOrder(apiKey, orderId.id) else Task.now(false)
      trade    <- getTrade(apiKey, order)
      balances <- getBalances(apiKey)
    } yield ExecutionDetails(trade, balances)
```

Our execution method returns an `ExecutionDetails` structure, with an optional `Trade` object if the order was filled, and the new account balances. 

If you look closely at the above implementation you'll see that we haven't defined the `getOrderStatus` method. This is where FS2 comes it. What we do is we poll the API every so often, until we finally give up. If we give up, we then cancel the order before finishing up and returning. 

This is a simple task for FS2. The following code will get the job done. 

```scala
def getOrderStatus(orderId: String, verifyFrequency: FiniteDuration, verifyAttempts: Long): Task[Order] =
  fs2.Stream
    .eval(Task.schedule(getOrder(apiKey, orderId)), verifyFrequency)
    .repeat
    .takeThrough(_.state == "PENDING")
    .take(verifyAttempts)
    .runLog
    .map(_.last)
```

If you know FS2, this code is dead straightforward. If you're not familiar with FS2, the steps are as follows:

1. `eval(Task.schedule(getOrder(apiKey, orderId)), verifyFrequency)` execute a task afer delay of `verifyFrequency`. The result is emitted to the stream.
2. `repeat` tells it to do exactly that, indefinitely.
3. `takeThrough(_.state == "PENDING")` tells the stream to continue emitting elements as long as `_.state == "PENDING"`, but also emit the first element in which this condition fails.
4. `take(verifyAttempts)` tells the stream to emit at most `verifyAttempts` elements.
5. `runLog` "runs" the stream and generates a vector output, and
6. `last` returns the last element of the vector, an `Order`.

Note that we've put "runs" in quotes, as it doesn't really run the stream, it just transforms the stream into a `Task` that returns a vector when the task is run.

So this is very simple code - far simpler than the handcrafted imperative equivalent, or some kind of recursive scheme.

So now we have our order execution code, and now we want to make some money from it. But hang on a second, we haven't tested it.


