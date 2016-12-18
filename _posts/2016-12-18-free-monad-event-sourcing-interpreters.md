---
layout: post
slug: free-monad-event-sourcing-interpreters
title: Interpreters for Event Sourcing Free Monads
date: 2016-12-08
---

This post follows from the post on [Free Monads and Event Sourcing Architecture]({% post_url 2016-12-08-free-monads-and-event-sourcing %}). The event sourcing interpreter implementation was a bit vague, and we will develop it in this post.

To summarise the implementation so far, restricted to the command side of the equation, suppose we have a an algebra `CommandOp` and a `publishEvent` method that can publish events of type `CommandOp[A]`.  

We then have an interpreter wrapper function:

```scala
final case class CommandCapture[F[_], G[_]](interpreter: F ~> G) extends (F ~> G) {
  override def apply[A](fa: F[A]): G[A] = {
    publishEvent(fa)
    interpreter.apply(fa)
  }
}
```

Here the `F` type constructor represents the free algebra, in this case `CommandOp`, and `G` represents the target monad, which could be something like `Task`.

The given our original interpreter `commandInterpreter: CommandOp ~> Task`, we create a wrapped interpreter:

```scala
val capturingInterpreter = CommandCapture(commandInterpreter)
```

Then when we run our program as follows:

```scala
val commandTask = program.foldMap(capturingInterpreter)
val result = commandTask.unsafeRun()
```

There are significant problems with this approach. 

One of the main purposes of the `Task` monad is to delay the execution of effectful code until the latest possible stage. Any code that has effects should be wrapped in a `Task` object, and the processing of these tasks should happen in the execution of `Task.unsafeRun()`. The code that generates this `Task` object should itself be pure, and have no effects. When this practice is adhered to, we can rely on it that any code that is not executed `Task.unsafeRun()` is referentially transparent, and can be much more easily reasoned about. And in particular any code that isn't wrapped doesn't involve `Task` at all, can be assured to be pure. `Task` becomes a containment zone for impure code.

The problem here is `publishEvent` is effectful code, and inserting it here violates these principles. 

This may sound quite abstract, but to point out a specific concrete consequence of this, note that in `val commandTask = program.foldMap(capturingInterpreter)` a task is simply created, and no effect should have taken place. But in doing so, we have already logged the operation! But just because you create a task, it doesn't mean you need to take it further. It's fully your prerogative to decide not to execute it, or to execute it several times. In both these cases, we would have logged the operation exactly once. In the latter case, it shouldn't make any difference whether the event is inserted once or many times, due to the idempotence condition of the event sourcing implementation, but it's still clearly flawed.

Now that we understand the problem, what can we do to improve it?

Firstly, it's worth noting that most free monad interpreters are effectful, and quite by definition, any useful interpreter of a "command" algebra is effectful. As described above, it's a good practice to seek to delay the processing of these effects to a final end step. As a consequence we may have several layers of interpreters, where each one is a natural transformation from one monad to another. The final transformation would be to a monad such as `Task`, who's purpose it is to process these effects. We can represent the chained sequence of transformations by the type `CommandOp ~> Task`. 

A desireable property of our event sourcing implementation in this case is that it also only performs the event logging operations as part of running a `Task` instance.

For various reaons however, there are some cases where we may want to use another effect processing monad, such as `Future`. In these cases we also want the event sourcing implementation to adhere to this, and do the actual logging of events by executing a `Future`.

So we can distill the problem to the following general case: 

Given a free algebra `F`, and natural transformation `F ~> M` to a monad `M`, how do we create an augmented natural transformation `F ~> M` that allows us to capture the events in `F` but only process them when we process `M`. This is what we are going to derive.

The first step in the solution is practice what we preach, and abstract the process of logging events into its own free algebra. This algebra only needs a single Append event.

```

```








