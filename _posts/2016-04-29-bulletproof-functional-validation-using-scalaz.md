---
layout: post
slug: bulletproof-functional-validation-using-scalaz
title: Bulletproof Functional Validation using Scalaz
date: 2016-04-29
---

As a backend developer, you are going to want to expose your efforts through an API such as a REST API. Your API may be publicly exposed, which means you can't even trust that the caller has good intentions. Good validation of the incoming parameters is essential. However it is not that easy to achieve with conventional imperative code, at least not in a way that is not very messy and doesn't obscure the intent of the code itself. We investigate a functional paradigm for validation, step-by-step, using the Scalaz `ValidationNel` applicative functor. As an example, we apply it to a Play REST endpoint.

Consider a Play webservice with a single endpoint, taking 6 parameters. Here is the `routes` file:

```scala
GET  /search  controllers.SearchController.search(
                keywords: String, 
                topLeftLat: BigDecimal, 
                topLeftLon: BigDecimal, 
                bottomRightLat: BigDecimal, 
                bottomRightLon: BigDecimal, 
                searchMethod: Option[String])
```

Let's start with an implementation without validation:

```scala
package controllers

case class GeoPoint(lat: BigDecimal, lon: BigDecimal)

case class BoundingBox(topLeft: GeoPoint, bottomRight: GeoPoint) 

class SearchRepository {
  def search(keywords: String, searchMethod: String, boundingBox: BoundingBox): Future[JValue] = ???
}

class SearchController @Inject() (repo: SearchRepository)(implicit ec: ExecutionContext) extends Controller {

  def search(keywords: String,
               topLeftLat: BigDecimal,
               topLeftLon: BigDecimal,
               bottomRightLat: BigDecimal,
               bottomRightLon: BigDecimal,
               searchMethod: String) = {

      Action.async { request: Request[AnyContent] ⇒
        val topLeft = GeoPoint(topLeftLat, topLeftLon)
        val bottomRight = GeoPoint(bottomRightLat, bottomRightLon)
        val boundingBox = BoundingBox(topLeft, bottomRight)
        val listings: Future[JValue] = repo.search(keywords, searchMethod, boundingBox)
        listings map { json ⇒ Ok(pretty(json)) }
      }
    }
}
```

## An *imperative* for validation

This is all nice and clean. Now we wish to add validation. Initially, the validation we require is as follows:

* Latitudes must be between 90 and -90 (inclusive).
* Longitudes must be between -180 and 180 (inclusive).
* `topLeftLat` must be greater than `bottomRightLat`.
* `topLeftLon` must be less than `bottomRightLon`.
* The `searchMethod` must be an acceptable value.

This is just the start. There are several other validations we might apply to tighten it up further.

In terms of output we require:

* If validation is successful an `OK` (`200`) HTTP response must be returned.
* If validation fails an `BAD_REQUEST` (`400`) HTTP response must be returned.
* The `search` controller method returns a future with this response.

Here is our first attempt at validation, just to see how ugly it can get:

```scala
def search(keywords: String, 
        topLeftLat: BigDecimal, 
        topLeftLon: BigDecimal, 
        bottomRightLat: BigDecimal,
        bottomRightLon: BigDecimal, 
        searchMethod: String) = {

    if (topLeftLat < -90 || topLeftLat > 90) {
        Future.successful(badRequest("topLeftLat must be between -90 and 90")) 
    } else {
        if (bottomRightLat < -90 || bottomRightLat > 90) {
            Future.successful(badRequest("bottomRightLat must be between -90 and 90")) 
        } else {
            Action.async { request: Request[AnyContent] ⇒
                val topLeft = GeoPoint(topLeftLat, topLeftLon)
                val bottomRight = GeoPoint(bottomRightLat, bottomRightLon)
                val boundingBox = BoundingBox(topLeft, bottomRight)
                val searchMethod = getListingSearchMethod(searchMethodOption)
                val listings: Future[JValue] = repo.search(keywords, searchMethod, boundingBox)
                listings map { json ⇒ Ok(pretty(json)) }
            }
        }
    }
}
```

That's probably a good time to give this approach up as a bad idea. We've only validated the range of the lattitudes, and we're already 2 nested levels of `if` statements deep before we get to the meat of it. We would have avoided the nesting by returning straight out of the function with the required error value. That would have been acceptable in Javaland, but it has problems of its own, and is frowned upon as a practice in Scala. This is unaccepable; it's a bread-and-butter action, and we want these sort of things to be done in idiomatic Scala. 

There are other issues with this approach.

* Is is clear that this approach is not scalable with respect to the number parameters we have, nor the complexity of the validations we need to make.
* Some of the validation logic may be conditional. Whateven constitutes a valid parameter may depend on the value of other parameters. The result of this is that we may need to duplicate logic regarding processing the parameters, once for the validations, and then again for the "real work" itself.
* This will fail on the first invalid parameter. If there is more than one invalid parameters, we will only find out about the next after we fix the first.

Another option, still in the imperative paradigm, may be an mutable list of errors which gets accumulated as we go. This would avoid the nested ifs, and address some of the issues, but it is still quite ugly. 

Stepping back, what are requirements for good validation:

* It should be exhaustive and easy to follow where it is present and where it is not.
* It should not interfere or obscure the main intent of the operation.
* It should not be necessary to duplicate any parameter logic to accommodate validation.
* We should easily be able to share validation code across methods with similar parameters.
* Ideally we would like validation to be cumulatative:- we accumulate all the errors at once, and report back to the caller all the reasons why validation failed. This is particularly important for form validation.

## Starting with the familiar *Option*

Let's start with something on familiar ground, the [`Option` monad](http://stackoverflow.com/questions/25361203/what-exactly-makes-option-a-monad-in-scala), and work from there, even though it will be inadequate in obvious ways. 

```scala
def isValidLat(lat: BigDecimal): Boolean = ???
def isValidLon(lon: BigDecimal): Boolean = ???
def isValidSearchMethod(method: String): Boolean = ???

def search(keywords: String,
         topLeftLat: BigDecimal,
         topLeftLon: BigDecimal,
         bottomRightLat: BigDecimal,
         bottomRightLon: BigDecimal,
         searchMethod: String) = {

Action.async { request: Request[AnyContent] ⇒
  val action: Option[Future[Result]] = for {
    tlLat ← if (isValidLat(topLeftLat)) Some(topLeftLat) else None
    tlLon ← if (isValidLon(topLeftLon)) Some(topLeftLon) else None
    brLat ← if (isValidLat(bottomRightLat)) Some(bottomRightLat) else None
    brLon ← if (isValidLon(bottomRightLon)) Some(bottomRightLon) else None
    sm ← if (isValidSearchMethod(searchMethod)) Some(searchMethod) else None
    topLeft = GeoPoint(tlLat, tlLon)
    bottomRight = GeoPoint(brLat, brLon)
    boundingBox = BoundingBox(topLeft, bottomRight)
  } yield {
    repo.search(keywords, sm, boundingBox) map { json ⇒ Ok(pretty(json)) }
  }
  action match {
    case Some(thingTodo)    ⇒ thingTodo
    case None               ⇒ Future.successful(BadRequest)
  }
}
}
```

We've added some validation methods, and wrapped the logic in a for comprehension. We only unwrap it at the end to execute the `search` method if all the parameters are valid, and to return a `BadRequest` if validation fails. This already goes a long way to the solution. We've managed to avoid all nested `if` statements, and avoid obfuscating the code. But the obvious problem with it is that we have no way of knowing what went wrong if validation does fail.

## Or we could *Either* improve things 

Our next step is to try out the `Either` monad from the Scala standard library.

```scala
class Either[+E, +A]
case class Left[+E, +A](e: E) extends Either[E, A]
case class Right[+E, +A](a: A) extends Either[E, A]
```

The one way of thinking about `Either` is like a labeled `Option`. Specifically, `Right` is analogous to `Some` and `Left` is a labeled `None`. Think or it as a branching structure where normal execution takes us `Right` and unexpected execution takes us `Left` (think of the Latin for right and left, *dexter* and *sinister* respectively, if this helps to remember). So we can use the `Left` to store what went wrong. All the usual operations, like `map` and `flatMap` operations operate on the `Right`, and do nothing to the `Left`.

```scala
def search(...) = {
    Action.async { request: Request[AnyContent] ⇒
      val action: Either[String, Future[Result]] = for {
        tlLat ← if (isValidLat(topLeftLat)) Right(topLeftLat): Either[String, BigDecimal] else Left("Invalid latitude")
        tlLon ← if (isValidLon(topLeftLon)) Right(topLeftLon): Either[String, BigDecimal] else Left("Invalid latitude")
        ...
      } yield {
        repo.search(keywords, searchMethod, boundingBox) map { json ⇒ Ok(pretty(json)) }
      }
      ...
    }
}
```

## Ramping up to Scalaz

`Either` is a big improvement on the `Option` based solution. As a small wrinkle, the type hint `Either[String, BigDecimal]` is needed for it to build. But it works well. In fact, Scalaz provides a purpose built structure with minor modifications, and a few bells and whistles:

```scala
sealed abstract class Validation[+E, +A]

case class Success[A](a: A) extends Validation[Nothing, A]
case class Failure[E](e: E) extends Validation[E, Nothing]
```

We can use this exactly as with `Either`, but with cleaner syntax, where extension methods `success` and `failure` are provided:

```scala
  val action: scalaz.Validation[String, Future[Result]] = for {
    tlLat ← if (isValidLat(topLeftLat)) topLeftLat.success else "Invalid latitude".failure
    tlLon ← if (isValidLon(topLeftLon)) topLeftLon.success else "Invalid latitude".failure
    ...
  } yield {
    repo.search(keywords, searchMethod, boundingBox) map { json ⇒ Ok(pretty(json)) }
  }
  action match {
    case Success(method) ⇒ method
    case Failure(errors) ⇒ badRequest(errors)
  }
```

The syntax is a little cleaner, the type hints are not necessary, but otherwise it works exactly the same. However it generates the following compiler warning:

```scala
Warning:(42, 18) method ValidationFlatMapDeprecated in object Validation is deprecated: flatMap does not accumulate errors, use `scalaz.\/` or `import scalaz.Validation.FlatMap._` instead
        tlLat <- if (isValidLat(topLeftLat)) topLeftLat.success else "Invalid latitude".failure
                 ^
```

This give an indication as to why we are not there yet. The for comprehension syntax, which involves nested `map`s and `flatMap`s under the hood, cannot acumulate errors. This is because if the moment a validation fails, the inner `map` function does not get called. This behaviour is baked into the `flatMap` function, and `for` comprehension behaviour, as each validated value is in scope for the statement that follows. Instead we consider an alternative functional mechanism, for which an instance is provided by `Validation`, the *applicative functor*. This turns out to be more suitable by nature for solving the problem at hand. Just as `map` and `flatMap` are the fundamental method of functors and monads respectively, the fundamental method of an applicative functor is `apply2` &#42;. `apply2` is a extension of `map` to a second dimension. It is like a `map` on two separate objects at once, where the inner contents of the two objects are formed into a pair, and a function is applied to this pair. If `map` were to represent a select on a single database table, `apply2` would represent a join. For an applicative functor `F[_]`, `apply2` has this signature:

```scala
def apply2[A, B, C](fa: ⇒ F[A], fb: ⇒ F[B])(f: (A, B) ⇒ C): F[C] 
```

** &#42;Actually a applicative functor is defined as structure with the following method, which is equivalent to `apply2` - consider this an exercise!**

```scala
def ap[A,B](fa: => F[A])(f: => F[A => B]): F[B]
```

We start by transforming the familiar `for` comprehension syntax:

```scala
val action = for {
    a ← validateA
    b ← validateB
    c ← validateC
} yield {
    doSomething(a, b, c)
}
```

into the slightly less familiar syntax:

```scala
val action = (
    validateA |@| 
    validateB |@| 
    validateC
    ) {
    (a, b, c) ⇒ doSomething(a, b, c)
}
```
Not as expressive as the original for comprehension syntax, but still elegant enough. In the latter case, all three `validate` operations are called, regardless of whether the other operations fail or not, and `a` is clearly not in scope for `validateB` and `validateC`. Only if they all succeed is `doSomething` is called, and if any of them fail, this action is bypassed, and the list of errors is returned. There are instances where we can only validate a parameter once we have a valid value of another parameter. For example, we expect that `bottomRightLat > toLeftLon`. For such cases we can use `flatMap` / `for` comprehensions, and in practice we will end up with a combination of `for`s and `|@|`s. If we are using `for`s we may want to include `import scalaz.Validation.FlatMap._` as suggested by the compiler warning. 

Given the availability of an applicative functor on the `Validation` object, the `|@|` operator is a convenient bit of Scalaz shorthand wizardry that instantiates this applicative functor, and invokes the `apply2` method on it. It's even smart enough to combine chained `|@|` operations into calls to `applyN`.

How are the errors accumulated? The "left" type parameter of `Validation` must have a `SemiGroup` instance. That's a fancy way of saying that it has an append or concatenate operation at its disposal. This append operation technically must be *associative*, which itself is a fancy way of saying that if we want to concatenate the 3 objects, we can combine the first two, and then add the third at the end, or we can combine the last two and add then add the first one to the front, and either way it will end up the same. The implementation of `apply2` will just invoke the semigroup `append` function to accumulate failures. The implementation is not exactly this, but this is the sense of it:

```scala
def apply2[EE, A, B, C](valA: Validation[EE, A], valB: Validation[EE, B])(f: (A,B) ⇒ C)(implicit E: Semigroup[EE])
    : Validation[EE, C] = (valA, valB) match {
    case (Success(a), Success(b))       ⇒ Success(f(a,b))
    case (e @ Failure(_), Success(_))   ⇒ e
    case (Success(_), e @ Failure(_))   ⇒ e
    case (Failure(e1), Failure(e2))     ⇒ Failure(E.append(e2, e1))
}
```

The most useful concrete case for the left parameter type is a `List` (which is clearly a `SemiGroup`), and specifically a non-empty list, as `Failure(List())` does not make any sense. So useful is it, that there is a special Scalaz type, `ValidationNel`, with some of it's own additional features. 

```scala
type ValidationNel[+E, +X] = Validation[NonEmptyList[E], X]
```

This is what we end up using. We can rewrite our controller code as follows:

```scala
  val action: scalaz.ValidationNel[String, Future[Result]] = (
    tlLat ← if (isValidLat(topLeftLat)) topLeftLat.successNel else "Invalid latitude".failureNel |@|
    tlLon ← if (isValidLon(topLeftLon)) topLeftLon.successNel else "Invalid latitude".failureNel |@|
    ...
  ) { (tlLat, tlLon, ..., sm) ⇒ 
    val topLeft = GeoPoint(topLeftLat, topLeftLon)
    val bottomRight = GeoPoint(bottomRightLat, bottomRightLon)
    val boundingBox = BoundingBox(topLeft, bottomRight)
    repo.search(keywords, sm, boundingBox) map { json ⇒ Ok(pretty(json)) }
  }
  action match {
    case Success(method)                ⇒ method
    case Failure(NonEmptyList(errors))  ⇒ badRequest(errors)
  }
```

`successNel` and `failureNel` are syntactic sugar extension methods. For example, `"Invalid latitude".failureNel` is the same as `Failure(NonEmptyList("Invalid latitude"))`. 

We aren't completely done, but we've broken the back of it. We may want to tidy up our code further by wrapping our errors in a case class. For example, for form validation we may have:

```scala
case class InvalidField(fieldName: String, reason: String)
```

and then define a new validation type:

```scala
type FieldValidation[A] = ValidationNel[InvalidField, A]
```

In addition we may wish to factor out some of our validation code into new methods. For example:

```scala
private def getValidBoundingBox(topLeftLat: BigDecimal, 
                                topLeftLon: BigDecimal, 
                                bottomRightLat: BigDecimal, 
                                bottomRightLon: BigDecimal): FieldValidation[BoundingBox] = ???
```

Our final robust validation solution couldn't be much simpler:

```scala
  Action.async { request: Request[AnyContent] ⇒
    val action = (getValidKeywords(keywords) |@|
                  getValidSearchMethod(searchMethod) |@|
                  getValidBoundingBox(topLeftLat, topLeftLon, bottomRightLat, bottomRightLon)) {
        (validKeywords, validMethod, boundingBox) ⇒
          repo.search(validKeywords, validMethod, boundingBox) map toJsonString(request)
      }
    action match {
      case Success(method) ⇒ method
      case Failure(errors) ⇒ badRequest(errors)
    }
  }
```

The great thing is we can factor out validation method as we like, and all validation errors get hoovered up and stored in the order they occurred, and the main action method only gets executed if all validation succeeds. If there are validation errors, the response to the client may look like this:

```json
{
   "validationErrors":[
      {
         "field":"topLeftLat",
         "reason":"Must be between -90.0 and 90.0 (inclusive)"
      },
      {
         "field":"topLeftLon",
         "reason":"Must be between -180.0 and 180.0 (inclusive)"
      },
      {
         "field":"bottomRightLat",
         "reason":"Must be between -90.0 and 90.0 (inclusive)"
      },
      {
         "field":"searchMethod",
         "reason":"Must be one of: ['name', 'category', 'tag']"
      }
   ]
}
``` 

If this walkthrough isn't able to validate (excuse the pun) the benefits of functional programming, I can't imagine what will. This is not some esoteric, isolated use case. Input validation is something *every* production application has to do, and lots of it. Before functional programming came along, or at least before I started using it, I was never able to find a satisfactory way of doing this.







