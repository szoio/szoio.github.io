---
layout: post
slug: free-monad-event-sourcing-interpreters
title: Interpreters for Event Sourcing Free Monads
date: 2016-12-08
---

Our one final topic is event playback. Playback involves reading from the event store, and creating a program in our free algebra that invokes the commands that have been stored. This then gets intrepreted as usual. A key point here is that the playback mechanism is completely separate from the interpretation, and unlike for recording, where we interleave command instructions with instructions to record these, it doesn't
need to care what this interpreter is. So in this sense it is much simpler.

Furthermore the generation of these instructions is closely aligned to the implementation details of the event recording interpreter.

So there isn't really a problem to tackle in it's full generality. For this reason we only look at a specific example, and in this case based on the recording interpreter above, we consider playing back the event log from a SQL database.

The complexity in playback is more performance related as we may be restoring a considerably large dataset. For this we use FS2.














 


