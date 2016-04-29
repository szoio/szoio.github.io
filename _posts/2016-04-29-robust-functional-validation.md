---
layout: post
slug: robust-functional-validation
title: Robust Functional Validation using Scalaz
date: 2016-04-29
---

As a backend developer is going to want to expose your efforts through an API, say a REST API. Your API may even be publicly exposed, which means you can't even trust that the caller has good intentions. Good validation of the incoming parameters is essential. However it is not that easy to achieve with conventional imperative code, at least not in a way that is very messy and obscures the intent of the code itself. We look at a functional paradigm for validation, using the Scalaz `ValidationNel` applicative functor. As an example, we apply it to a Play REST endpoint.

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

class SearchController @Inject() (repo: SearchRepository)(implicit ec: ExecutionContext) extends Controller {

    def search(keywords: String, 
            topLeftLat: BigDecimal, 
            topLeftLon: BigDecimal, 
            bottomRightLat: BigDecimal,
            bottomRightLon: BigDecimal, 
            searchMethodOption: Option[String]) = {

        Action.async { request: Request[AnyContent] ⇒
            val topLeft = GeoPoint(topLeftLat, topLeftLon)
            val bottomRight = GeoPoint(bottomRightLat, bottomRightLon)
            val boundingBox = BoundingBox(topLeft, bottomRight)
            val searchMethod = getListingSearchMethod(searchMethodOption)
            val listings: Future[JValue] = repo.searchListings(keywords, searchMethodOption, boundingBox)
            listings map { jsonString ⇒ Ok(pretty(jsonString)) }
        }
    }
```

This is all nice and clean. Now we wish to add validation. The validation we require is as follows:

* Latitudes must be between 90 and -90 (inclusive)
* Longitudes must be between -180 and 180 (inclusive)
* `topLeftLat` must be greater than `bottomRightLat`
* `topLeftLon` must be less than `bottomRightLon`

In addition we require:

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
            searchMethodOption: Option[String]) = {

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
                    val listings: Future[JValue] = repo.searchListings(keywords, searchMethodOption, boundingBox)
                    listings map { jsonString ⇒ Ok(pretty(jsonString)) }
                }
            }
        }

```

That's probably a good time to give this approach up as a bad idea. We've only validated the range of the lattitudes, and we're already 2 nested levels of `if` statements deep before we get to the meat of it. We would have avoided the nesting by returning straight out of the function with the required error value. That would have been acceptable in Javaland, but it has problems of its own, and is frowned upon as a practice in Scala. This is unaccepable; it's a bread-and-butter action, and we want these sort of things to be done in idiomatic Scala.

There are other issues with this approach.

* Is is clear that this approach is not scalable with respect to the number parameters we have, nor the complexity of the validations we need to make.
* Some of the validation logic may be conditional. Whateven constitutes a valid parameter may depend on the value of other parameters. The result of this is that we may need to duplicate logic regarding processing the parameters, once for the validations, and then again for the "real work" itself.
* This will fail on the first invalid parameter. If there is more than one invalid parameters, we will only find out about the next after we fix the first.

So what are requirements for good validation:

* It should be exhaustive and easy to follow where it is present and where it is not.
* It should not interfere or obscure the main intent of the operation.
* It should not be necessary to duplicate any parameter logic to acommodate validation.
* Ideally we would like validation to be cumulatative:- we accumulate all the errors at once, and report back to the caller all the reasons why validation failed. This is particularly important for form validation.














