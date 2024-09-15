+++
title = "ZIO-HTTP and the ETag header"

date = 2024-09-15
+++

# Introduction

In this article, I will showcase a simple HTTP application that uses the zio-http library.
First, we will look at the business problem, implement a naive solution, then add some optimization with ETag headers.

The code is available [here](https://github.com/kurgansoft/etag-example).

Let's get started!

# First steps

Let's say we want to create a mobile app that allows users to place their orders in restaurants using their phones.

In order to display a nice-looking UI that can be used to create an order, first, we need to fetch the menu.

So, let's start by modeling the menu:

``` scala
case class Catalogue(version: Int, items: Map[String, Int])
```

The Catalogue class is quite minimal; it only contains the available items with their prices in a Map, and the version. For a real implementation, we would probably need more details, but it is good enough for our example.

Also, we need to generate a schema in the companion object. (More on that later.)

``` scala
object Catalogue {
  implicit val schema: Schema[Catalogue] = DeriveSchema.gen
}
```

As a next step, we will create a [declarative endpoint](https://zio.dev/zio-http/reference/endpoint) which will be used to fetch the latest catalogue:

```scala
val getCatalogue: Endpoint[Unit, Unit, ZNothing, Catalogue, None] = 
    Endpoint(RoutePattern.GET / "catalogue")
        .outCodec(StatusCodec.Ok ++ HttpCodec.content[Catalogue])
```

The above represents a GET endpoint with path 'catalogue' that responds with 200 and returns a JSON object. 
Because we have created the schema for the Catalogue class, we'll be able to use Catalogue scala objects in our implementation, the conversion to JSON will be taken care of automatically.

# Implementing the server

So let's implement the server so we can use our favorite http client to retrieve the catalogue!

The implementation that transforms this Endpoint to a Route object:

``` scala
private[server] val getCatalogueRoute: Route[Ref[Catalogue], Nothing] = 
  EndpointDefinitions.getCatalogue.implement(_ =>
    for {
      ref <- ZIO.service[Ref[Catalogue]]
      currentCatalogue <- ref.get
    } yield currentCatalogue
  )
 ```

 We are using a [Ref](https://zio.dev/reference/concurrency/ref/) here to get access to the current Catalogue.
 It also means that later we'll need to provide this environment to the Route before we can use it.

So we'll need an effect that periodically updates this Ref with the latest Catalogue object. To have some sample data I have asked ChatGPT to generate 25 food items with their prices in HUF. This is the result:

``` scala
val hardcodedItems: Seq[(String, Int)] = Seq(
    "Pizza Margherita" -> 1500,
    "Cheeseburger" -> 1200,
    "Spaghetti Carbonara" -> 1800,
    "Caesar Salad" -> 1600,
    "Club Sandwich" -> 1400,
    "Chicken Soup" -> 800,
    "Grilled Sirloin Steak" -> 3500,
    "Fish and Chips" -> 2200,
    "Sushi Platter" -> 4000,
    "Tonkotsu Ramen" -> 2500,
    "Chicken Curry" -> 1900,
    "Beef Tacos" -> 1200,
    "Loaded Nachos" -> 1300,
    "Charcuterie Board" -> 3000,
    "Gelato Trio" -> 900,
    "Chocolate Lava Cake" -> 1100,
    "Apple Pie" -> 900,
    "Bruschetta" -> 800,
    "Garlic Bread" -> 700,
    "Goulash" -> 1700,
    "Beef Stroganoff" -> 2600,
    "Pierogi" -> 1300,
    "Ratatouille" -> 1500,
    "Fried Chicken" -> 2500,
    "Vegetable Stir Fry" -> 1600
)
```

The idea is that we'll have an effect that updates this Ref every half second with a new Catalogue object derived from the above dataset.

```scala
private val updateCatalogueEffect: ZIO[Ref[Catalogue], Nothing, Unit] = for {
    catalogueRef <- ZIO.service[Ref[Catalogue]]
    _ <- ZIO.foreachDiscard(1 to CatalogueGenerator.size) {
      round =>
        for {
          _ <- catalogueRef.set(CatalogueGenerator.data(round))
          _ <- ZIO.sleep(Duration.fromMillis(500))
        } yield ()
    }
  } yield ()
```

Link to [CatalogueGenerator](https://github.com/kurgansoft/etag-example/blob/master/src/main/scala/etag_demo/server/CatalogueGenerator.scala).

After 25 iteration the Catalogue object will no longer be updated, so we will need one more endpoint to reset this process:

```scala
val reset: Endpoint[Unit, Unit, ZNothing, Unit, None] = 
  Endpoint(RoutePattern.POST / "reset")
    .out[Unit].outCodec[Unit](HttpCodec.empty)
```

Now we can implement the server:

```scala
override def run: ZIO[Scope, Throwable, Unit] = for {
    catalogueRef <- Ref.make(CatalogueGenerator.data(1))
    updateFiberId <- updateCatalogueEffect.provideEnvironment(ZEnvironment(catalogueRef)).fork
    fiberIdRef <- Ref.make(updateFiberId)
    routes = Routes(getCatalogueRoute, resetRoute).provideEnvironment(ZEnvironment(catalogueRef).add(fiberIdRef))
    _ <- Server.serve(routes).provide(Server.default)
    _ <- ZIO.never
  } yield ()
```

I have omitted the implementation of the resetRoute, but you can check it out [here](https://github.com/kurgansoft/etag-example/blob/a44710784f48e47a5e42c55fed9f94fee2937ab2/src/main/scala/etag_demo/server/Main.scala).
We are creating a Routes object from our two routes, providing the missing dependencies then launching the Server with default parameters.

# Invoking the endpoint

At this stage our application is working, we can launch Main.scala. I'm using [Bruno](https://www.usebruno.com/) to do some exploratory testing.

We can use the reset endpoint to start with catalogue version 1:

![](/reset.png)

Then depending on when we invoke the GET endpoint we get back a catalogue with a version between 1 and 25:

![](/get.png)

# Creating a client

At the beginning of this article, we mentioned that we need to fetch the catalogue to display a UI. If we have retrieved it already, it is possible that it is not the latest version.

We have defined our endpoints with the high level Endpoint API, and with that it is quite straightforward to use those endpoints from a client.

We just need to import the Endpoint definitions:

```scala
import etag_demo.common.EndpointDefinitions.{getCatalogue, reset}
```

Then we can create an endpoint executor:

```scala
executor <- ZIO.service[EndpointExecutor[Any, Unit]]
```

And with the executor we can call the endpoints in RPC style:

```scala
_ <- executor(reset())
latestCatalogue <- executor(getCatalogue())
```

The first line returns Unit, while the second one yields a Catalogue. No need for manual conversions from HTTP responses, or parsing JSON manually.

Below is the extract from [Client1](https://github.com/kurgansoft/etag-example/blob/master/src/main/scala/etag_demo/client/Client1.scala) which retrieves the Catalogue 250 times and checks how many times we received the same value we already had:

```scala
object Client1 extends ZIOAppDefault {

  override def run: ZIO[Scope, Unit, Unit] = (for {
    executor <- ZIO.service[EndpointExecutor[Any, Unit]]

    _ <- ZIO.log("calling reset endpoint")
    _ <- executor(reset())

    catalogueRef <- Ref.make[Option[Catalogue]](None)
    noOfBandwidthWastingCalls <- Ref.make[Int](0)

    _ <- ZIO.foreachDiscard(1 to 250)(_ => for {
      latestCatalogue <- executor(getCatalogue())
      _ <- ZIO.log("The latest catalogue retrieved: " + latestCatalogue)
      currentCatalogue <- catalogueRef.get
      _ <- if (currentCatalogue.exists(_.version == latestCatalogue.version))
        noOfBandwidthWastingCalls.update(_ + 1) *> ZIO.log(s"Seems like we have fetched catalogue with version [${latestCatalogue.version}], but we had that one already.")
      else
        catalogueRef.set(Some(latestCatalogue))
      _ <- ZIO.sleep(zio.Duration.fromMillis(50))
    } yield ())

    noOfBandwidthWastingCallsAsNumber <- noOfBandwidthWastingCalls.get
    _ <- ZIO.log(s"\n\tWe have executed $noOfBandwidthWastingCallsAsNumber bandwidth wasting calls.")
  } yield ()).provide(
    EndpointExecutor.make(serviceName = "server").orDie,
    Client.default.orDie,
    Scope.default,
  )
}
```

In order to run the above successfully we need to set the **SERVER_URL** env variable like **SERVER_URL=http://localhost:8080**.

After a successful run, we will have the following message:


>  We have executed 225 bandwidth wasting calls.

It is no surprise since we have 25 version of the catalogue so we can only retrieve a new version at most 25 times. So 250-25=225 times we just retrieve the version that we already had.

# Fixing the problem with ETag

The previous approach solves the problem, but not in an efficient way. We could use Server-Sent Events, but that is a topic for another day.

What we would like to do is to tell the server the version that we already have, and in case it did not change we get a reply that it was not modified and an empty body. In case it is not the current version, we will retrieve that just as before.

We can achieve the above using the [If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match) request header and the [ETag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag) response header.

So let's create a new endpoint definition with those two:

```scala
val getCatalogueWithETag: Endpoint[Unit, Option[Header.IfNoneMatch], ZNothing, Either[Header.ETag, (Catalogue, Header.ETag)], None] =
  Endpoint(RoutePattern.GET / "catalogueWithETag")
    .inCodec(HeaderCodec.ifNoneMatch.optional)
    .outCodec(StatusCodec.Ok ++ HttpCodec.content[Catalogue] ++ HttpCodec.header(Header.ETag))
    .outCodec(StatusCodec.NotModified ++ HttpCodec.header(Header.ETag))
```

So what has changed compared to the previous endpoint? Now we have an input value: **Option[Header.IfNoneMatch]**, which means - no surprise - it can optionally take an If-None-Match header.
We are using the outCodec method two times, that is why the response type is an Either. The first represents a reply with a Catalogue object and an **ETag** header, while the second one the **ETag** header alone.

We can think about the ETag as a hash function for the resource. In this case we will just use the version number as the ETag value.

Server implementation for the new endpoint:

```scala
private[server] val getCatalogueRouteWithETag: 
  Route[Ref[Catalogue], Nothing] = 
    EndpointDefinitions.getCatalogueWithETag.implement(nonMatchHeader =>
      for {
        ifNonMatchValue <- nonMatchHeader match {
          case Some(headerValue) =>
            zio.Console.printLine("If-None-Match header is present: " + headerValue).orDie.as(Some(headerValue.renderedValue))
          case _ => zio.Console.printLine("If-None-Match header is not present.").orDie.as(None)
        }
        ref <- ZIO.service[Ref[Catalogue]]
        currentCatalogue <- ref.get
        s304 = ifNonMatchValue.contains(currentCatalogue.version.toString)
        _ <- zio.Console.printLine("s304: " + s304).orDie
        etag = Strong(currentCatalogue.version.toString)
        toReturn = if (s304) Left(etag) else Right((currentCatalogue, etag))
        _ <- zio.Console.printLine("About to reply with: " + toReturn).orDie
      } yield toReturn
    )
```

We just check if the **If-None-Match** header is present, and if it is, it has the same version number that the actual Catalogue object has. If yes, we return with a Left value, otherwise with a Right.

# Client that sends the ETag value

Now, we can create a new Client that uses this new endpoint. We're doing pretty much the same thing as before, but now we are holding on to the previous ETag value and using it in subsequent requests. You can check the implementation [here](https://github.com/kurgansoft/etag-example/blob/master/src/main/scala/etag_demo/client/Client2.scala).

After executing it we get the message:

> We have saved some bandwidth 225 times.

# Testing

We have done some exploratory testing with an HTTP client, and written a client app for both the endpoints. But it would be nice to have a test that we can run on CI and shows that our endpoints behave the way they should.

Let's focus on our second endpoint, more precisely the Route we create from it.
So what do we want to assert? Basically three things:
* when request DOES NOT contain header 'If-None-Match' - we always get back the actual content
* when request DOES contain header 'If-None-Match' and it is the same as the Etag of the response - we get back 304 and empty body
* when request DOES contain header 'If-None-Match' but it is different than the Etag of the response - we get back 200 and the actual body

In each of those cases we don't really care what the content is - we just want to get that back. In other words we can say that if the server has a given content, we want the described behavior.

Therefore we can utilize [property based testing](https://zio.dev/reference/test/property-testing/). First we create a [Generator](https://zio.dev/reference/test/property-testing/how-generators-work) for the Catalogue class:

```scala
def mapGenerator(numberOfItems: Int): Gen[Any, Map[String, Int]] = for {
  items <- Gen.listOfN(numberOfItems)(Gen.alphaNumericStringBounded(1,5))
  prices <- Gen.listOfN(numberOfItems)(Gen.int(500, 20000))
} yield items.zip(prices).toMap

val catalogueGen: Gen[Any, Catalogue] = for {
  version <- Gen.fromIterable(1 to 10)
  numberOfItems <- Gen.fromIterable(1 to 10)
  map <- mapGenerator(numberOfItems)
} yield Catalogue(version, map)
```

The above can generate Catalogue objects with version numbers between 1 and 10 and they can contain 1 to 10 items. Prices are between 500 and 20000, and the item names are just random alphanumeric characters.

Let's see the implementation for the second test case:

```scala
val routesUnderTest = Main.getCatalogueRouteWithETag.toRoutes

test("when request DOES contain header 'If-None-Match' and it is the same as the Etag of the response - we get back 304 and empty body") {
  check(catalogueGen)(catalogue => {
    val req = Request.get(URL(Path.root./("catalogueWithETag"))).addHeader(Header.IfNoneMatch.parse(catalogue.version.toString).getOrElse(???))
    for {
      ref <- Ref.make(catalogue)
      app = routesUnderTest.provideEnvironment(ZEnvironment(ref))
      response <- app.runZIO(req)
    } yield assertTrue(response.status == Status.NotModified && response.body.isEmpty)
  })
}
```

With the check function we generate a Catalogue object and then we create a request that contain the version of that object in the **If-None-Match** header. Later we inject that object wrapped in Ref into the routes we are testing. Then we can execute our request and assert two things:

* status code is 304
* response body is empty

Later we annotate the test suite with 

```scala
TestAspect.samples(1000)
```

so we can check the above behavior with 1000 Catalogue objects.

[Here](https://github.com/kurgansoft/etag-example/blob/master/src/test/scala/etag_demo/server/RouteTest.scala) is the complete implementation with the other two scenarios.

# Conclusion

In this article, we explored high-level endpoints in zio-http, discussing their implementation on the server side and how to generate clients utilizing this technology.

We employed ETags to enhance our polling solution, effectively reducing bandwidth usage.

There are a couple of things we haven't looked at, like ETag in non-GET requests, sending multiple ETag values, 
and understanding the difference between strong and weak ETags.
But hopefully you have managed to learn something, and adopt some ideas in your solution.



