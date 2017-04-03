---
layout: post
slug: free-monad-event-sourcing-playback
title: Free Monad Event Sourcing - Playback and Wrap-up
date: 2017-03-30
---

In this post, following on from [Free Monad Event Sourcing Interpreters]({% post_url 2016-12-18-free-monad-event-sourcing-interpreters %}), we complete the series on event sourcing with free monads by discussing playback. We demonstrate by example, using the [doobie data access library](https://github.com/tpolecat/doobie) and the [FS2 streaming library](https://github.com/functional-streams-for-scala/fs2) to playback an event log of operations recorded and serialised as Json in a SQL database. There isn't that much to say about it, but we've included this post for the sake of completeness. Recording is only half of the story, in this case the more difficult half, but it's still worth discussing playback. If you are not familiar with FS2, it will also serve as a good motivating example for FS2 and its usage.

Playback involves reading from the event store, and creating a program in our free algebra that invokes the commands that have been stored. This then gets interpreted as usual. A key point here is that the playback mechanism is completely separate from the interpretation, and unlike for recording, where we interleave command instructions with instructions to record these, it doesn't need to care what this interpreter is. So in this sense it is much simpler than recording. 

Playback is somewhat implementation specific - there isn't really a problem that can be dealt with in any kind of generality. For this reason we rather consider a specific example. Following on from the recording interpreter presented [before]({% post_url 2016-12-18-free-monad-event-sourcing-interpreters %}), we consider playing back the event log from a SQL database, using [doobie](https://github.com/tpolecat/doobie) for data access.

The complexity in playback is mainly performance related, as we may be restoring a considerably large dataset. 

For playback we have the following requirements:

* Constant memory complexity
* Linear time complexity 
* Playback must be in the same order as the recorded events

Here complexity is in terms of the size of the event log.

For this a stream such as that provided by FS2 is a great fit. Actually FS2 is a great fit for many other completely different use cases, and that may be the subject of a future post or two.

In terms of the implementation, it's really quite straightforward, and much of the recording mechanism can be reused. We only need to add some analogous code to decode and convert back from the serialised to the instructions of our original algebra.

```scala
trait Event2M extends EventSourcing with EventInterpreter {
  ...
  def e2f[A](fa : E) : F[A]           // Conversion from event to command
  def decoder : Decoder[E]

  private val streamAll: Stream[ConnectionIO, Event] =
    sql"select id, created, payload from event".query[Event].process

  def playbackM(transactor : Transactor[M], chunkSize : Int)(f2M : F ~> M)(implicit M : Monad[M]) : Stream[M, M[Unit]] = {
    val conIOStream : Stream[ConnectionIO, Free[F, Unit]] =
      streamAll
        // convert the Json into an event
        .map { dbEvent => decoder.decodeJson(dbEvent.payload).toOption }
        // discard events that fail to decode
        .collect { case Some(x) => x }
        // convert the events into operations of type Free[F, Unit]
        .map { event => Free.liftF[F, Any](e2f(event)).map(_ => ()) }

    val batchedStream : Stream[ConnectionIO, Free[F, Unit]] =
      conIOStream.chunkLimit(chunkSize)
        // sequence the chunk to convert to a single Free instance (and convert to Unit)
        .map(_.toList.sequence[Free[F, ?], Unit].map(_ => ()))

    // transform stream from ConnectionIOs to Ms
    val fStreamM : Stream[M, Free[F, Unit]] =
    transactor.transP(batchedStream)

    // interpret the stream of `Free[F, ?]`s to create a stream of `M`s
    fStreamM.map(_.foldMap(f2M))
  }
}
```

Here we have added a playback method that reads the event log table and returns a `Stream[M, M[Unit]]`. We break this down into steps:

1. Create a `Stream[ConnectionIO, Event]`
  
  This is entirely a doobie task, using the `process` method to turn the results of a query into a FS2 stream.

2. Transform this stream into a `Stream[ConnectionIO, Free[F, Unit]]`
  This is the decoding step, where all events gets deserialised into the original free monad type. Here the exact type inference rules for Scala get a little murky, as we don't know the exact type of the event, but by lifting it using `Free.liftF[F, Any]` and then mapping to `Unit` we get around this.

3. Convert the stram into batches, or chunks, which are then packaged together into sequences of free instances. This is an optional step, for efficiency only. If the target database that is being reconstituted from playback is a SQL database, it may be necessary, as otherwise every playback step will generate its own transaction, which might slow things down somewhat.  

4. Transform effect monad from `ConnectionIO` to the target effect monad `M`. This operation represents the operation of reading the database via a JDBC connection, but of course as is always the case, the effect of actually doing this is postponed for later. This gives us a steram of `Stream[M, Free[F, Unit]]`.

5. Interpret the free monads emitted from the stream to generate a stream of `M`s. Not that the target effect monad appears in both parameters.

Running the stream requires a choice of target monad `M`. In the case of `Task`, we can run it in the natural way using something like

```scala
val playbackStream: Stream[Task, Task[Unit]]

playbackStream.map(_.unsafeRun).unsafeRun
```

or one of the "safer" `Task` run variants. Note as usual that this is the only effectful step. All other steps before are just transforming values that represent sequences of computations.

If you are using free monads extensively, you may end with free monads at various levels of abstraction in your system. It is obviously not necessary to apply event sourcing to each of them. We only need to record the events at the level with is required to recostitute system state. It makes a lot of sense to apply event sourcing to the free monad at the lowest level of abstraction in your domain model, above any level the level of abstraction concerned directly with data access. Events should have meaning in the domain model rather than reveal or record implementation details of your system, something that would commit you to that implementation. On the other hand, recording events at too high a level of abstraction, the events themselves may end up being quite brittle, and you may miss certain events if this level is bypassed.

This completes the series on Free Monads and event sourcing. I hope after reading this series, you feel inspired to apply this pattern in any of the projects you are working on.

All of this is available in the following [gist on Github](https://gist.github.com/szoio/b80a5c5fb8da00be5a2e5fd822b7895e).

