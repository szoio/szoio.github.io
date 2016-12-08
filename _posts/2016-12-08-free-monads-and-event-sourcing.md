---
layout: post
slug: free-monads-and-event-sourcing.md
title: Free Monads and Event Sourcing Architecture
date: 2016-12-08
---

Since delving into free monads over the last 6 months, they have become ubiquitous in our code base. Free monads is a functional programming pattern which you describe the instructions that a program in a way that is completely separated from the execution of these instructions. 

But hang on, doesn't that sound like an interface in object oriented programming? Yes, there are similarities. But they are differences some pretty major differences, such as:

1. Free monads are closed under monadic operations, in the mathematical sense. Just like a vector space is closed under linear operations. This means that any set of monadic operations, such as maps, flatmaps as well as extension such as traverses can be performed on a free monad, and it still free monad - it retains its essential nature. Monadic operations are by no means all-encompassing, but are often rich enough and sufficiently expressive to span the full domain of what they are intended for. You certainly can't say anything like that about interfaces in OO! 

2. Free monads are data. They are sometimes referred to as a *"program that describes a program"*, but the actually *"a data structure that represents a program that describes a program"*. 

Free monads are a quite a difficult concept to get your head around, but they are easy to work with. This blog is not intended as a comprehensive description on free monads. There are quite a few resources for that:- here is quite a nice free reading description of free monads:
http://perevillega.com/understanding-free-monads
There's also many excellent talks available online, and of course [Red functional programming book](https://www.manning.com/books/functional-programming-in-scala) is a classic and always to be recommended, and describes the pattern from first principles.

Instead the point of this post is to relate free monads to [*event sourcing*](http://martinfowler.com/eaaDev/EventSourcing.html), another well known and very worthwhile pattern. The principle of event sourcing relates to data storage and management. It suggests a different way of thinking about your data in which the current database state is not the fundamental source of truth, but the ordered sequence of events that were used to get it into this state. Databases are generally mutable, and their state is modified by create, update and delete operations (everything but the read in the so called CRUD operations). This sequence of events that describe these mutations are themselves immutable, and together constitute an append-only event log. Event sourcing is treating this event log as the source of truth, and the database as a projection or snapshot in time of the cumulative history of events.

There are numerous benefits to event sourcing, including but not limited to:
1. *Audit trail* - having a complete sequence of timestamped events enables you to rewind your database state to any point in time.
2. *Robustness* - if there's any failure in the database, or data is not committed or otherwise lost, you can fall back on the event log and try again, or restore from there.
3. *Migrations* - data migrations can sometimes be very difficult, especially if it involves moving to an entirely different data model. Event sourcing simplifies this and diminishes the risk. The new database is just inflated by applying the events to the new data model or database instance. And it can be done in real time with no downtime. 

Free monads and event sourcing are an excellent match. So much so, that after understanding the benefits of free monads, and following some of them to their natural conclusion, you'd end up inventing event sourcing if it didn't exist already.

A term that if often associated with event sourcing is CQRS (Command query responsibility segregation). That's a complicated way of saying separating commands (create, update and delete operations) from queries (read operations). This concept is important and 

Free monads enable you to simplify your thinking about a programming task. When creating a system, you consider the domain you are trying to model, and you come up with an instruction set to model that domain. In functional programming speak, that is often referred to as an algebra, I suppose for the closure reason given above. So to borrow Martin Fowler's example, we could create a simple language or algebra from the following events:

* Departure: ship leaves a port
* Arrival: ship arrives at a port
* Load: cargo is loaded on a ship
* Unload: cargo is unloaded from a ship

```scala
sealed trait ShipLocation
case object AtSea extends ShipLocation
case class InPort(placeCode: PlaceCode) extends ShipLocation

sealed trait ShipOp[A]

object ShipOp {
  // Commands
  case class AddPlace(name: String, placeCode: PlaceCode) extends ShipOp[Unit]
  case class AddShip(name: String, shipCode: ShipCode) extends ShipOp[Unit]
  case class Departure(shipCode: ShipCode, placeCode: PlaceCode, time: DateTime) extends ShipOp[Unit]
  case class Arrival(shipCode: ShipCode, placeCode: PlaceCode, time: DateTime) extends ShipOp[Unit]
  case class Load(shipCode: ShipCode, placeCode: PlaceCode, time: DateTime) extends ShipOp[Unit]
  case class Unload(shipCode: ShipCode, placeCode: PlaceCode, time: DateTime) extends ShipOp[Unit]

  // Queries
  case class GetLocation(ship)
}

```





