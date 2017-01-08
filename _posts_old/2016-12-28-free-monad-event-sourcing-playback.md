---
layout: post
slug: free-monad-event-sourcing-playback
title: Free Monad Event Sourcing - Playback and Wrapu
date: 2016-12-31
---

In this post, following on from [Free Monad Event Sourcing Interpreters]({% post_url 2016-12-18-free-monad-event-sourcing-interpreters %}), we complete the series on event sourcing with free monads by discussing playback. We demonstrate by example, using the [doobie data access library](https://github.com/tpolecat/doobie) and the [FS2 streaming library](https://github.com/functional-streams-for-scala/fs2) to playback an event log of operations recorded and serialised as Json in a SQL database.

Playback involves reading from the event store, and creating a program in our free algebra that invokes the commands that have been stored. This then gets interpreted as usual. A key point here is that the playback mechanism is completely separate from the interpretation, and unlike for recording, where we interleave command instructions with instructions to record these, it doesn't need to care what this interpreter is. So in this sense it is much simpler than recording. 

Furthermore the generation of these instructions is closely aligned to the implementation details of the event recording interpreter.

So there isn't really a problem that can be tackled in any kind of generality. For this reason we rather consider a specific example. Following on from the recording interpreter presented [before]({% post_url 2016-12-18-free-monad-event-sourcing-interpreters %}), we consider playing back the event log from a SQL database.

The complexity in playback is more performance related, as we may be restoring a considerably large dataset. 

For playback we have the following requirements:

* Constant memory complexity
* Linear time complexity 
* Playback must be in the order of the recorded

where complexity is in terms of the size of the event log.

For this a stream is a prefect fit. Much of the recording mechanism can be reused. We only need to add some analogous code to decode and convert back.

```
trait EventSourcing {
  // Abstract methods
  ...
  ...
}


trait Event2M extends EventSourcing with EventInterpreter {
  ...
  def e2f[A](fa : E) : F[A]           // Conversion from event to command
  def decoder : Decoder[E]

  def playback(transactor: Transactor[M], chunkSize: Int)(f2M: F ~> M)(implicit M: Monad[M]) : Stream[M, M[Unit]] = {
    val conIOStream : Stream[ConnectionIO, Free[F, Unit]] =
      spring.marketintel.eventlog.pg.queries.event.streamAll
        // convert the Json into an event
        .map { dbEvent => decoder.decodeJson(dbEvent.payload).toOption }
        // discard events that fail to decode
        .collect {  case Some(x) => x }
        // convert the events into operations of type Unit
        .map { event => Free.liftF[F, Any](e2f(event)).map(_ => ()) }

    val batchedStream : Stream[ConnectionIO, Free[F, List[Unit]]] =
      conIOStream.chunkLimit(chunkSize)
        // sequence the chunk to convert to a single Free instance
        .map(_.toList.sequence[Free[F, ?], Unit])

    // transform stream from ConnectionIOs to Ms
    val fStreamM : Stream[M, Free[F, List[Unit]]] =
      transactor.transP(batchedStream)

    // interpret the stream of `Free[F, ?]`s to create a stream of `M`s
    val mStreamM : Stream[M, M[List[Unit]]] =
      fStreamM.map(_.foldMap(f2M))

    // simplify the return type to `Unit`
    mStreamM.map(_.map(_ => ()))
  }

}

```












 


