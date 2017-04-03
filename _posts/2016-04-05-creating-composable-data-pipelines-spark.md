---
layout: post
slug: creating-composable-data-pipelines-spark
title: Composing Spark data pipelines
date: 2016-04-05
---

We investigate a pattern of functional programming that we can apply to [Apache Spark](http://spark.apache.org/) to help create clean and elegant code for data transformations, and avoiding state management headaches. We do so by separating the definition of data transformation processes from their execution. 

[Apache Spark](http://spark.apache.org/) is a marvellous piece of software engineering that has taken the big data world by storm. It is currently the most active Scala project and the most active big data project, with a large community of contributors. There are plenty of resources available for finding out more about and learning Spark, and the docs are excellent. This post focuses on some areas of deficiency. These are not necessarily deficiencies in the Spark framework itself, but rather pitfalls that could result in your Spark code getting a little messy without careful planning. This post suggests a pattern that could avoid many of these.

In going about this, we also describe a functional programming pattern which is also generally applicable in many similar contexts, known as the [Reader monad](http://2016.phillyemergingtech.com/2012/system/presentations/di-without-the-gymnastics.pdf). In spite of this pattern being widely useful, it's not actually that easy to find non-trivial examples which are still easy enough to understand. Hopefully this post will help somewhat. 

To demonstrate, we start off with the canonical example of a MapReduce task, a word count across a large corpus of documents. A simple Spark program may look like this:

```scala
import org.apache.spark.{ SparkConf, SparkContext }

object WordCountOneLiner {
  def main(args: Array[String]): Unit = {

    val sc = new SparkContext(
      new SparkConf().setMaster("local[2]").setAppName("WordCount"))

    sc.textFile(Util.baseDir() + "/TXT/*")      // Read text files
      .flatMap { line => line.split("\\W+") }   // Split lines into words
      .map(_.toLowerCase)
      .filter(!_.isEmpty) 
      .map((_, 1))                              // Create (word,1) pairs
      .reduceByKey(_ + _)                             
      .takeOrdered(100)(Ordering.by(-_._2))     // sort descending by count

    sc.stop()                                   // terminate SparkContext
  }
}
```

This really simple functional code illustrates much of the appeal of Spark, espacially for Scala developers and functional programmers in general.
All the operations that define the map and reduce steps are just ordinary scala functions. 

In order to do anything in Spark we need to create a `SparkContext` object. This object is a handle to access and manage the resources required for distributed data processing tasks. It needs to be stopped when finished with to free up resources on the cluster.

## Refactoring into a trait

Unfortunately, real world data programming tasks are never as simple as this simple word count. In addition, we may wish to test each step in the process separately. To cope with this, we refactor our one line code into a trait:

```scala
trait WordCountSimple {

  val sparkContext = new SparkContext(
    new SparkConf().setMaster("local[2]").setAppName("WordCount"))

  def lines = sparkContext.textFile(Util.baseDir() + "/TXT/*")

  def words = lines.flatMap { line => line.split("\\W+") }
    .map(_.toLowerCase)
    .filter(!_.isEmpty)

  def count = words.map((_, 1)).reduceByKey(_ + _)

  def topWords(n: Int) = count.takeOrdered(n)(Ordering.by(-_._2))

  def stop(): Unit = sparkContext.stop()
}
```

Immediately design problems become apparent:

* The `SparkContext` object that is part of the trait. What happens if we have several traits, each representing different data processing logic, and we wish to share the `SparkContext` across these traits?
* Even worse - the `stop()` method that needs to be called when we're done. Who is responsible for doing this, and when?

So without doing anything non-trivial, we already have a lifecycle and state management headache. 

What would help is if we could decouple the creation and management of the `SparkContext` from the transformation logic.

## Introducing the SparkOperation monad

```scala
sealed trait SparkOperation[+A] {
  // executes the transformations
  def run(ctx: SparkContext): A

  // enables chaining pipelines, for comprehensions, etc.
  def map[B](f: A => B): SparkOperation[B] 
  def flatMap[B](f: A => SparkOperation[B]): SparkOperation[B] 
}
```

and companion object:

```scala
import scalaz._

object SparkOperation {
  def apply[A](f: SparkContext => A): SparkOperation[A] = new SparkOperation[A] {
    override def run(ctx: SparkContext): A = f(ctx)
  }

  implicit val monad = new Monad[SparkOperation] {
    override def bind[A, B](fa: SparkOperation[A])(f: A ⇒ SparkOperation[B]): SparkOperation[B] =
      SparkOperation(ctx ⇒ f(fa.run(ctx)).run(ctx))

    override def point[A](a: ⇒ A): SparkOperation[A] = SparkOperation(_ ⇒ a)
  }
}
```

Firstly, to bring everyone onto the same page, what is a *monad*? A *monad* is a key abstration in functional programming. Whenever one is looking for a general solution to composability, monads are normally not too far away. By composability, we mean the output of one process is the input into another. And that is precisely what we are trying to do. A data processing pipeline consists of several operations, each joined together to form the pipeline. And we can join two or more pipelines to form larger pipelines. A monad has this property. Technically, in order to make this possible, a monad must have a `map` and a `flatMap` method. The `flatMap` method is the distinguishing feature of the monad abstractions as it is the operation the enables composability. If you aren't familiar with monads and their associated methods, it takes a little time to get comfortable with them.

In addition to these methods being present, some algebraic laws need to be satisfied. We don't go into these too deeply, but just mention that these laws simply ensure that things happens as we expect they should. An example is *associativity*. In this context this means that if we have pipelines *A*, *B* and *C*, and we wish to join them to form a single pipeline *ABC*, we can either join *A* and *B* and then join the result *AB* and *C*, or we could join *A* to pipeline *BC*. No need to worry too much about this, suffice to say, the `SparkOperation` satisfies these monad laws, and it's not too hard to verify this. Note that the `SparkOperation` trait is sealed. That means we don't allow other traits or classes to derive from it. This is because we would not be able to ensure that the derived classes still satisfy the same monad laws, and as a result may get used incorrectly.

If you are familiar with it, you may recognise `SparkOperation` as an example of a [Reader monad](http://2016.phillyemergingtech.com/2012/system/presentations/di-without-the-gymnastics.pdf). We rely on a bit of help from [scalaz](https://github.com/scalaz/scalaz) to form the monad. Rather than using Scalaz, we could have implemented `map` and `flatMap` as follows:

```scala
sealed trait SparkOperation[+A] {
  ...
  def map[B](f: A ⇒ B): SparkOperation[B] = 
  SparkOperation { ctx ⇒ f(this.run(ctx)) }
  def flatMap[B](f: A ⇒ SparkOperation[B]): SparkOperation[B] = 
  SparkOperation { ctx ⇒ f(this.run(ctx)).run(ctx) }
}
```
However, there are benefits to letting Scalaz provide the monad instance, as we will see in some of the functional compostion examples below. 

## Rewriting the pipeline using SparkOperations

Rewriting our trait in terms of `SparkOperation`s is a simple task.

```scala
trait WordCountPipeline {

  // initial SparkOperation created using companion object
  def linesOp = SparkOperation { sparkContext =>
    sparkContext.textFile(Util.baseDir() + "/TXT/*")
  }

  // after that we often just need map / flatMap
  def wordsOp = for (lines <- linesOp) yield {
    lines.flatMap { line => line.split("\\W+") }
      .map(_.toLowerCase)
      .filter(!_.isEmpty)
  }

  def countOp = for (words <- wordsOp) 
    yield words.map((_, 1)).reduceByKey(_ + _)

  def topWordsOp(n: Int): SparkOperation[Map[String, Int]] = 
    countOp.map(_.takeOrdered(n)(Ordering.by(-_._2)).toMap)
}
```

In general, the first operation in the data pipeline normally involves the companion object. That is because we can't start the process without a `SparkContext`. From then on, we often create don't need to explicitly refer to the `SparkContext`, and subsequent operations are created through `map`s, `flatMaps` and other functional operations.

To execute the pipeline operations, we could write the following code to get a word count of the top 100 words:

```scala
  val sc = new SparkContext(
    new SparkConf().setMaster("local[2]").setAppName("WordCount"))
  val topWordsMap: Map[String, Int] = topWordsOp(100).run(sparkContext)
  sc.stop()
```

The key design accomplishment is we have completely decoupled the definition of the processing logic from the execution of the pipeline. There is no reference to a `SparkContext` instance in the pipeline trait, so we can freely mix and match pipeline traits, take operations from one trait and use their output to form new operations in another trait without worrying about how they would share a `SparkContext`. 

Having this separation gives us a great deal of flexibilty regarding the actual execution. For example, we could execute the same process locally using a short lived `SparkContext`, or on a cluster on a long lived `SparkContext` (how we go about this will be explained in a future post).

## Some examples of functional composition

Here are some examples of how we can apply functional to create new operations from existing one. 

### Joins (applicative join operation)

A common operation is a join. Any Spark RDD on a pair `RDD[(K,A)]` has a join method:

```scala
def join[W](other: RDD[(K, B)], partitioner: Partitioner): RDD[(K, (A, B))]
```

We can lift this join operation into a `SparkOperation` using the `|@|` operator provided by Scalaz:


```scala
import scalaz.syntax.bind._

object JoinExample {
  trait K
  trait A
  trait B

  val opA: SparkOperation[RDD[(K,A)]] = ???
  val opB: SparkOperation[RDD[(K,B)]] = ???

  val joinedOp: SparkOperation[RDD[(K,(A,B))]] = 
    (opA |@| opA)((rddA,rddB) => rddA.join(rddB))
}
```

### Sequence operation composed from list

In this example, we turn an operation that performs a task for a given date to an operation that performs this task for a range of dates using the `sequence` operator.

```scala
import scalaz.std.list.listInstance

object SequenceExample {
  trait A

  val dateList: List[Date] = ???
  def opForDate(s: Date) : SparkOperation[A] = ???
  
  val opOfLists: SparkOperation[List[A]] = 
    SparkOperation.monad.sequence(dateList map opForDate)  
}
```

Using these patterns we can really clean, functional code that is simple to understand, maintain and extend. We also have a great deal of flexibility in how these jobs are executed, as nothing in the process definition says anything about the the process execution.

This reader monad pattern naturally extends to any framework or library where operations require an expensive context to be created beforehand. An good example is a database connection. A database connection monad of a similar nature can be used to simplify code for operations that require this connection. 

## The Sparkplug library

The functionality above is available in the [Sparkplug library](https://github.com/springnz/sparkplug). The library includes:

* The `SparkOperation` monad
* Testing tools for sampling and persisting test data
* Simple and efficient execution on a cluster

## Testing tools

Among the testing tools is a very handy set of extensions on `SparkOperation`s. Of particular interest is the extension method:

```scala
def sourceFrom(rddName: String, sampler: RDD[A] ⇒ RDD[A] = identitySampler)
```

This enables one to save down a data sample to disk. This is really helpful for creating a test data set from data sourced in a database instance. It's typically done by extending the pipeline trait, and overriding the data operation as follows:

```scala
import springnz.sparkplug.testkit._

trait WordCountTestPipeline extends WordCountPipeline {
  def linesOp = super.linesOp.sourceFrom("test_data_set", sampler)
}
```

The way this works if no test data is present, the original dataset will be used, passed through the sampling function, and the resultant condensed dataset is save into the test resource folder. Once the data is there, there will no longer be any call to the original datasource. 
`sampler` is a RDD sampling function, typically used to reduce the size of the dataset to something manageable for unit tests. A few of these are provided out the box, including a random sampler and sample that takes the first `n` records. 

So it is easy to create test data for unit tests without having any reliance on external data connections in your continuous integration environment. The idea is that if you do a `git pull` followed by `sbt test`, everything should just work.

I hope this has been useful, both in terms of giving you ideas about how you could get the most out of Apache Spark if you use it, and also how this functional pattern could be applied in other development scenarios that you might have.

Cluster execution strategies are covered in this post on [executing Spark jobs with Akka]({% post_url 2016-04-17-spark-cluster-execution-with-akka %}).







