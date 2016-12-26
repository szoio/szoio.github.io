---
layout: post
slug: free-monad-event-sourcing-interpreters
title: Interpreters for Event Sourcing Free Monads
date: 2016-12-18
---

This post follows from the post on [Free Monads and Event Sourcing Architecture]({% post_url 2016-12-08-free-monads-and-event-sourcing %}). Here we develop a fully generic event sourcing interpreter framwork for capturing events from any free algegbra. We describe a concrete example instance that uses the [doobie](https://github.com/tpolecat/doobie) library to persist the events to a SQL database. Our implementation is typesafe and performant. 

Even though our [implementation so far]({% post_url 2016-12-08-free-monads-and-event-sourcing %}) is vague, it is clearly evident that the approach taken is flawed. We summarise in the section following, restricting to the command side of the equation, and explain the shortcomings.

Suppose we have a an algebra `CommandOp` and a `publishEvent` method that can publish events (instructions of this type).  

We then have an interpreter wrapper function:

```scala
final case class CommandCapture[F[_], M[_]](interpreter: F ~> M) extends (F ~> M) {
  override def apply[A](fa: F[A]): M[A] = {
    publishEvent(fa)
    interpreter.apply(fa)
  }
}
```

Here the `F` type constructor represents the free algebra, in this case `CommandOp`, and `M` represents the target monad, which could be something like `Task`.

The given our original interpreter `commandInterpreter: CommandOp ~> Task`, we create a wrapped interpreter:

```scala
val capturingInterpreter = CommandCapture(commandInterpreter)
```

Then when we run our program as follows:

```scala
val commandTask = program.foldMap(capturingInterpreter)
val result      = commandTask.unsafeRun()
```

There are significant problems with this approach. 

One of the main purposes of the `Task` monad is to delay the execution of effectful code until the latest possible stage. Any code that has effects should be wrapped in a `Task` object, and the processing of these tasks should happen in the execution of `Task.unsafeRun()`. The code that generates this `Task` object should itself be pure, and have no effects. 

When this practice is adhered to, we can rely on it that any code that is not executed `Task.unsafeRun()` is referentially transparent, and can be much more easily reasoned about. And in particular any code that isn't wrapped doesn't involve `Task` at all, can be assured to be pure. `Task` becomes a containment zone for impure code.

The problem here is `publishEvent` is effectful code, and inserting it here violates these principles. 

This may sound quite abstract, but to point out a specific concrete consequence of this, note that in `val commandTask = program.foldMap(capturingInterpreter)` a task is simply created, and no effect should have taken place. But in doing so, we have already logged the operation! But just because you create a task, it doesn't mean you need to take it further. It's your prerogative to *not* execute it, or to execute it several times. In both these cases, we would have logged the operation exactly once. In the latter case, it shouldn't make any difference whether the event is inserted once or many times, due to the idempotence condition of the event sourcing implementation, but it's still far from ideal.

Now that we understand the problem, what can we do to improve it?

Firstly, it's worth noting that most free monad interpreters are effectful, and quite by definition, any useful interpreter of a "command" algebra is effectful. As described above, it's a good practice to seek to delay the processing of these effects to a final end step. Consequently we may have several layers of interpreters, where each one is a natural transformation from one monad to another. The final transformation would be to a monad such as `Task`, who's purpose it is to process these effects. We can represent the chained sequence of transformations by the type `CommandOp ~> Task`. 

A desireable property of our event sourcing implementation in this case is that it also only performs the event logging operations as part of running a `Task` instance.

For various reaons however, there are some cases where we may want to use another effect processing monad, such as `Future`. In these cases we also want the event sourcing implementation to adhere to this, and do the actual logging of events by executing a `Future`.

So we can distill the problem to the following general requirement: 

Given a free algebra `F` representing the command instructions, and natural transformation `F ~> M` to a monad `M`, how do we create an augmented natural transformation `F ~> M` that allows us to capture the events in `F` but only process them when we process `M`? This is what we are going to derive.

## Abstracting event logging with free monads

The first step in the solution is practice what we preach, and abstract the process of logging events into its own free algebra. This algebra only needs a single `Append` instruction.

```scala
sealed trait EventOp[A]

final case class Append(event: E) extends EventOp[Unit]  
```

Now, instead of interpreting from `F` to `M` directly using our `F ~> M` natural transformation we create another layer `F ~> Free[C, ?]` where `C` is the coproduct of our original algebra `F` and the event logging algebra `EventOp` (note that the natural transformation destination type must be a monad). Then we construct an interpreter from `C ~> M`. Once we have this, it is straightforward to piece these together to get our desired `F ~> M` algebra that is able to process event logging effects.

This may sound a bit arcane to anyone not accustomed to working with free monads, but we break it down below and explain in more detail.

*Note that the `?` in `Free[C, ?]` is not native scala syntax, but is enabled by the [kind projector plugin](https://github.com/non/kind-projector).*

Lets start with the interpreter from `F ~> Free[C, ?]`:

```scala
type C[A]  = Coproduct[F, EventOp, A]
type FC[A] = Free[C, A] 

val f2FC = new (F ~> FC) {
  def apply[A](f: F[A]): FC[A] = {
    for {
      _ <- Event(fa).inject[FC]
      x <- fa.inject[FC]
  } yield x
}
```

This is very simple. Every time we get an instruction of the form `F[A]`, we create a code fragment that consists of an instruction combined with an addtional instruction to log the instruction. We inject both these instructions into the free monad of the coproduct of the `F` and `EventOp` algebras.

Then if we start off with a program of type `Free[F, ?]` comprising a set of instructions in `F`, and interpret them using:

```scala
val program: Free[F, ?]
val loggingProgram = program.foldMap(f2FC)
```

`loggingProgram` will end up being a new program in the augmented free monad `F[C, ?]` consisting of instructions from our original `F` algebra interleaved with instructions to log these `F` instructions as events. 

Note that no processing of the event logging takes place. Compare this with the `CommandCapture` interpreter above.

Proceeding, we create an interpreter `C ~> M` to our target monad `M`. This is also straightforward. 

To do this, we need an interpreter `F ~> M`, which was a given in our setup, and we also need an interpreter to process the logging effects `EventOp ~> M`. We can then combine them with `f2M or e2M`. The `or` method on natural transformations takes instructions from `C`, and interprets `F` instrucions with `f2M` and `EventOp` instructions with `e2M`.

Then we can chain these interpreters as follows:

```scala
val program: Free[F, ?]
val m = program.foldMap(f2FC).foldMap(f2M or e2M)
```

The standard choice for `M` would be something like `Task`. We could then process with something like

```scala
program.foldMap(f2FC).foldMap(f2M or e2M).unsafeRun()
```

There are times when we may want to choose another effect processing monad other than `Task`. This detail is dependent on the overall architecture. 

A specific example is the case where both the application database and the event store are SQL relational databases. Further, suppose they are using the same data connection, and [doobie](https://github.com/tpolecat/doobie) is used for data access. In this case it would make sense to choose the `ConnectionIO` free monad as the target monad `M`. If we do it this way we get the additional benefit that both commits to the application database and writes to the event log are performed in the same database transaction.

In the general case this may not be possible as the event log could be a NoSQL database such as Cassandra, or something completely different like a Kafka topic, a choice that would be well suited to a microservices architecture. Here the eventlog also serves as a message queue.

If there are any failures, we always prefer the application database to fail before the event log fails. If the event log fails first, it may not be possible restore the application database to the correct state from the event log, something for which the converse always holds, provided that all writes to the application database are idempotent. This is something we must bear in mind when designing our interpreters and their execution patterns.

## Creating an event logging interpreter

There is one detail that we still need to take care of, and that is the type `E` in

```scala
final case class Append(event: E) extends EventOp[Unit]  
```

In addition we have not yet considered any concrete implementations for the interpreter. We deal with both these below.

What we want `E` to represent is a serialisable form of our command algebra `F`. Using this our events will be serialised and persisted in the event log. These days you need a decent reason to not choose Json as a serialisation format, at least not until you data volume is such that binary serialisation becomes imperative. For Json processing in a FP Scala stack, especially using [Cats](http://typelevel.org/cats/), [Circe](https://circe.github.io/circe/) is a good fit. 

The task of converting to and from Json is handled generically by `Encoder` and `Decoder` type classes. Other Json libraries work in similar ways using type classes of different names. In this case we need a way of passing the `Encoder` typeclass instance to the interpreter. This is not a Json specific requirement - converter typeclasses is the most suitable mechanism for handling encoding into any serialisation format.

This is all reasonably straightforward to do if we have a single free monad, but what if we have multiple algebras, and we want to several of them to be event sourced, or if we wanted to create an event sourcing library that can capture events from any of our free monad algebras in a generic way?

We start by asking the question if this is even possible, and it fortunately turns out that it is.

Our first attempt is as follows:

```scala
sealed trait EventOp[E,　A]

final case class Append[E](event: E) extends EventOp[E, Unit]  
```

We then propagate this new `EventOp` class through the rest of the stack. This means that each of the object that depend on this additional parameter. For example our coproduct becomes:

```scala
type C[E, A] = Coproduct[F, EventOp[E, ?], A]
```

Having this additional parameter allows us to pass in an `Encoder` type class instance into our interpreter in a generic way. This enables us to define an interpreter 

```scala
def e2M(implicit encoder: Encoder[E]) = new (EventOp[E,?] ~>　M) {
  def apply[A](eva: EventOp[E,A]): M[A] = {
    val json = eva.asJson // this requires an implicit Encoder[E] instance
    ... // do something with the Json, like save it to Db or publish to Kafka
  }
}
```

However, with this approach, we start to push the limits of Scala's type inference capabilities. In particular, we run into problems with partially applied types. The particular problem we encounter is [SI-2712](https://issues.scala-lang.org/browse/SI-2712), evidently addresses in Scala 2.11.9 (not yet released) and 2.12, but not helpful for those stuck on 2.11.8.

An alternative approach that proves to be more fruitful is to create a base trait with an abstract type, and create our interpreter layers within this trait.

```scala
trait EventSourcing {  self : EventInterpreter =>
  // Abstract types
  type M[_]   // The target monad
  type F[_]   // The command algebra type
  type E      // The event type

  // Abstract methods
  implicit def encoder : Encoder[E]   // Encoder instance for Json encoding
  def f2e[A](fa : F[A]) : E           // Convertion from command to event

  // Event algebra and single Append instruction
  sealed trait EventOp[A]
  final case class Append(event : E) extends EventOp[Unit]

  // Shorthand types
  type C[A]  = Coproduct[F, EventOp, A]
  type FC[A] = Free[C, A]

  
  val f2FC = new (F ~> FC) {
    def apply[A](fa : F[A]) : FC[A] = {
      for {
        _ <- Free.inject[EventOp, C](Append(f2e(fa)))
        x <- Free.inject[F, C](fa)
      } yield x
    }
  }

  def f2MLog(f2M: F ~> M)(implicit M: Monad[M]): F ~> M = new (F ~> M) {
    override def apply[A](fa : F[A]) : M[A] = f2FC(fa).foldMap(f2M or e2M)
  }
}

trait EventInterpreter { self: EventSourcing =>
  def e2M(implicit M: Monad[M]): EventOp ~> M  // The event logging interpreter
}
```

The base trait `EventSourcing` carries all the generic machinery, generalised over serveral dimensions.

We then provide a specific implementation for `EventInterpreter.e2M`. In most cases we would need to fix the target monad `M` by overriding the type definition with a specific type. In this example, using Doobie to persist to a SQL event log, we don't need to, because Doobie works with a `Transactor` that itself is abstracted over the target monad.

```scala
trait Event2M extends EventSourcing with EventInterpreter {
  
  // doobie transactor, provided later when we fix M
  def transactor: Transactor[M]

  // Encoder instance for Json encoding, provided later when we fix E
  def encoder : Encoder[E]

  // SQL insert query as a Doobie ConnectionIO free monad
  private def append(event: E): ConnectionIO[Int] =
    sql"insert into mi.event(payload) values (${encoder(event)})".update.run

  // Our implementation of the `e2M` event logging interpreter
  override def e2M(implicit M: Monad[M]): EventOp ~> M = new (EventOp ~> M) {
    override def apply[A](fa: EventOp[A]): M[A] = fa match {
      case Append(e) =>
        M.map(append(e).transact(transactor))(_ => ())
    }
  }
}
```

Finally we create a algebra specific implementation, utilising these base traits. 
Only at this stage do we fix the algebra `F[_]` and the event type `E`, and provide a few simple overrides.

```scala
object Command {
  sealed trait CommandOp[A] { self: CommandEvent => }
  sealed trait CommandEvent { self: CommandOp[_] => }

  // Commands
  final case class CommandStr(str: String) 
    extends CommandEvent with CommandOp[Unit]   
  final case class CommandInt(int: Int) 
    extends CommandEvent with CommandOp[Boolean]

  val commandEncoder: Encoder[CommandEvent] = semiauto.deriveEncoder[CommandEvent]
  val commandDecoder: Decoder[CommandEvent] = semiauto.deriveDecoder[CommandEvent]

  case class CommandActions(trans: Transactor[Task]) extends Event2M {
    override type F[A]      = CommandOp[A]
    override type E         = CommandEvent
    override type M[A]      = Task[A]
    override def transactor = trans

    override def encoder : Encoder[CommandEvent] = commandEncoder
    override def f2e[A](fa : CommandOp[A])       = fa.asInstanceOf[CommandEvent]
  }

  def loggingInterpreter(trans: Transactor[Task])(f2T: CommandOp ~> Task)
    : CommandOp ~> Task =
    CommandActions(trans).f2MLog(f2T)
}

```

Simply with `Command.loggingInterpreter` we can now convert any interpreter from a `CommandOp` to a `Task` into an enhanced interpreter that simulatanously logs these events to a SQL database in the execution of the task. This is the solution to our original problem. 

Some observations:

* Our command events derive from two traits, `CommandOp[_]` and `CommandEvent`. The reason we require the `CommandEvent` in the first place is the Circe automatic encoder derivation only works for sealed trait families of case classes where the base trait is a concrete type, not a type constructor.
* We need a mechanism to convert from a `CommandOp` to a `CommandEvent` (and vice versa for playback). The `f2e` method does this. In our implementation we are doing an `asInstanceOf` cast, which is normally considered bad practice, but having these traits requiring each other using `{ self: CommandEvent => }` etc. ensures that this cast will not fail.

One last piece in the event sourcing puzzle is event playback. We'll briefly discuss this in a follow up post.









 


