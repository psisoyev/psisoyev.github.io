---
layout: post
title: Upgrade your Tagless Final with Tofu
date: 2020-12-18 13:37:00 +0100
description: This post introduces Tofu toolkit to Scala developers
img: train-station/trains-tofu.png
tags: [scala, apache pulsar, neutron, zio, tagless final, fs2, tofu]
comments: true
hidden: true
---

In my previous blog post, I've created an event-driven system built on top of Apache Pulsar.
The system was developed in Scala using the Tagless Final technique.
In this article, I would like to introduce you to a cool kid on the block that you might not know yet.
Meet [Tofu](https://github.com/TinkoffCreditSystems/tofu) - a functional programming toolkit aimed at taming the complexity of Tagless Final approach.
I will show how you can use utilities from Tofu to improve your codebase.
However, I must warn you that some of the concepts might seem advanced and look complex at the first sight but are actually pretty simple when you have a closer look.

In order to compare code improved with Tofu with traditional Tagless Final application, I've cloned the [train-station](https://github.com/psisoyev/train-station) repository.
We will use the same application to improve it with Tofu and then we will be able to compare the results.
If you are like me and prefer reading the code first and then read the article jump into the [new repo](https://github.com/psisoyev/train-station-tofu).

In this article I will show you how to:
* clean up your business logic from boilerplate;
* improve error handling;
* split runtime effect from application initialization effect;
* enable context-aware tracing;
* enable context-aware logging;
* enable JSON formatted logging;
* ???

I can't say that you *must* use all of that in your application as you and your team can already have built your own habits and style. 
My personal feeling is that Tagless Final is a great way of describing your application, however, sometimes the current ecosystem lacks some tooling. 
The idea of this post is to show you how to improve this but it doesn't mean it will solve all your problems.
You can use only part of the toolkit to solve particular problems. For that, the toolkit is split into several sub-modules. 
More info on this you can find in the [README file](https://github.com/TinkoffCreditSystems/tofu#quick-start).

The toolkit has much more to offer you:
* Agent - `Ref` on steroids, which allows effectful updates;
* Granular type classes for forking and racing;
* Env - a monad that allows composition of functions that are context-aware;
* Optics;
* Many other small but very useful utilities.

An important thing to mention is that the toolkit is not a `cats`/`cats-effect`/`Monix`/`ZIO`/`YOURIO` killer. 
I can't say something like "I'm using tofu stack". There is no "tofu stack". 
The idea of the toolkit is to improve your Tagless Final code for whatever effect runtime system you have.

# %Mid title% ???
The first thing we will see today is how to clean up application business logic from cross-cutting concerns surrounding the core of the logic.
For this, we will use a class called `Mid`.
It might slightly remind you of aspect-oriented programming.  
The idea is very simple - we extract utility logic from the core logic.
For example, logging or input data validation can be extracted into small separate modules. 

We will start with cleaning up the `Departures` service which we have created in the [previous article](https://scala.monster/train-station/).
If you haven't read the article or already have forgotten what it is about, you always can [find the service implementation on GitHub](https://github.com/psisoyev/train-station/blob/master/service/src/main/scala/com/psisoyev/train/station/departure/Departures.scala#L31).
I can see at least 2 things we could separate from the core logic.
The first one is logging before and after the method call.
The second one is the input data validation. 

Let's first agree on what is the core implementation logic of departure registration.
In our case, it is a generation of a random UUID and the creation of an event, which we return as the result of registration.
Let's extract it to a separate class and call it `Impl` (or `Core`, if you wish):
```scala
class Impl[F[_]: Applicative: GenUUID](city: City) extends Departures[F] {
  override def register(departure: Departure): F[Departed] =
    F.randomUUID.map { id =>
      Departed(
        EventId(id),
        departure.id,
        From(city),
        departure.to,
        departure.time,
        departure.actual.toTimestamp
      )
    }
}
```
That's the juice, the very sweet extract of the logic.
The application can work only having this and the rest is not important.
Ok, validating input data is also important, but we could survive without it.
We can clearly see the required [context bounds](https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html) for the implementation:
* `GenUUID` to generated unique id;
* `Functor` to apply a function on the result.

As now we have a separate class with the core logic, let's create classes for our "not so important" logic.  
We start with logging:
```scala
class Log[F[_]: FlatMap: Logger] extends Departures[Mid[F, *]] {  
  override def register(departure: Departure): Mid[F, Departed] = { registration =>
    val before = F.info(s"Registering $departure")
    val after  = F.info(s"Train ${departure.id.value} successfully departed")

    before *> registration <* after
  }
}
```
Here we require `FlatMap` to chain effectful function calls and a logger to actually do the logging.
If you have read the code carefully you could notice that now we extend the `Departures` trait not with just the effect type as we did with the core logic, but with `Mid` -
`extends Departures[Mid[F, *]]`. Also, the return type of registration is not a simple event wrapped in `F` but is a `Mid`.
As for a casual user, we won't even notice a difference.
Again we have to override `register` method. This time as we use `Mid`, we receive a new input - the result of the `registration` of type `F[Departed]`.
We basically surround the resulting effect with two other effects - a log before and a log after the registration.
The ice-cream symbols `*>` and `<*` are just symbolic aliases to `flatMap` method which drops the output.
Aand that's it. We don't need anything else here for logging.

Similarly to `Log` class we create one for validation:
```scala
class Validate[F[_]: Monad: DepartureError.Raising](
  connectedTo: List[City]
) extends Departures[Mid[F, *]] {
  def register(departure: Departure): Mid[F, Departed] = { registration =>
    val destination = departure.to.city

    connectedTo
      .find(_ === destination)
      .orRaise(UnexpectedDestination(destination)) *> registration
  }
}
```
Here we add a new context-bound which we haven't seen before - `DepartureError.Raising`.
We will have a closer look at it later when we will be talking about error management.
The class definition is similar to the logging class we saw before - we extend the `Departures` trait with `Mid` as the effect.
However, this time, the only thing we need is to do run validation checks before calling the core logic.
So we run our validation logic and call `registration` method `orRaise` an error.
If an error will be raised, then the core logic won't get called.

The only missing thing is gluing all the pieces together. 
We will use the same `make` method we had before, where we will initialize the classes and wire everything together. 
```scala
def make[F[_]: Monad: GenUUID: Logger: DepartureError.Raising](
    city: City,
    connectedTo: List[City]
): Departures[F] = {
  val service = new Impl[F](city)

  val log      = new Log[F]
  val validate = new Validate[F](connectedTo)

  (log |+| validate).attach(service)
}
```
First, we create an instance of core logic class, then instances for logging and validation classes.
We attach utility classes to the core service with a special method `attach`. 
Utility classes are combined with a special symbolic alias `|+|` which you could see in other libraries, like `cats`.
If you are not the biggest symbolic alias fan, then you could simply call `combine` method.

Note, that the order of running `Mid`s is deterministic.
In the code above, we have `log |+| validate`.
Logging is the outer wrapper and validation is the inner. 
It means that the order of the execution will be the following:
```
logging_pre
validation_pre
action
validation_post
logging_post
```

We are missing the one last thing - we can't combine (`|+|`) `Mid` instances without an implicit `ApplyK` in the scope.
This sounds complicated and might scare some people away.
Luckily, the easiest way to get an instance of it is to simply add an annotation on the original interface:
```scala
@derive(applyK)
trait Departures[F[_]] {
```
This `derive` annotation comes from another cool library called [`derevo`](https://github.com/manatki/derevo). 
The purpose of this library is various instance derivation using a single macro annotation.

Now the code compiles and we can be happy about having cleaner business logic. 
Here is the [link to the final code](https://github.com/psisoyev/train-station-tofu/blob/master/service/src/main/scala/com/psisoyev/train/station/departure/Departures.scala#L24).
Yes, the final version has slightly more lines of code, but it's a low price to pay for the clean code.

Some people might be not convinced with the example above because they don't log inputs and outputs.
Also, it's fine to have the validation service as a separate class.
Actually, this is what we did with the `Arrivals` service, as its validation is much bigger and potentially could grow even more.
However, the `Mid` concept can be used even in the validation service as well. [Have a look](https://github.com/psisoyev/train-station-tofu/blob/master/service/src/main/scala/com/psisoyev/train/station/arrival/ArrivalValidator.scala#L20).
There are more ideas what you could extract to `Mid`:
* tracing;
* authentication;
* authorisation;
* monitoring;
* caching;
* persistence;
* event publishing;
* and even more.

Another bonus of having core logic extracted is that it's possible to substitute the logic itself without changing the plumbing around.
It means that logging, monitoring, tracing, and all the other utilities will stay as-is.

# Improved error handling
Error handling is always a great topic for a holy war on the Internet.
Everyone knows how to do it in the best way, but everyone does that differently.

Tofu also provides a way to handle business errors. As an example, we will take care of `Departure` validation.
Above, we've already seen a new context bound in the `Departures` service - `DepartureError.Raising`.
It is signaling that this service can raise a `DepartureError`.
We've added it in the `DepartureError` companion object:
```scala
sealed trait DepartureError extends NoStackTrace
object DepartureError {
  type Handling[F[_]] = Handle[F, DepartureError]
  type Raising[F[_]]  = Raise[F, DepartureError]
  
  case class UnexpectedDestination(city: City) extends DepartureError
}
```
Here we also add a `Handling` type alias, which will be used as a context-bound in services that must handle `DepartureError`.

How do we use it? Easy, we already did it in the code above:
```scala
connectedTo
  .find(_ === destination)
  .orRaise(UnexpectedDestination(destination)) *> registration
```
We look for a train destination city in the `List` of connected cities.
The result of this search is an `Option`.  
In the import section, we add syntactic sugar import `import tofu.syntax.raise._`.
It contains the method `orRaise` which we call on `Option`.
If it is empty, we will raise an error, that we will have to handle later on.
There are other convenience methods to raise an error in different situations that can be found [here](https://github.com/TinkoffCreditSystems/tofu/blob/4337b6370ab0e7251ba87c02d1901e7d82f65b6b/core/src/main/scala/tofu/syntax/error.scala#L43).
In my opinion, the easiest way to learn all the syntactic sugar is by reading the code.

Of course, using syntactic sugar is not mandatory. There is always an option to call method `raise` on your effect as we did with `map` of `flatMap`:
`F.raise(UnexpectedDestination(destination))`. It is also typed and would expect a `DepartureError`. 

Handling errors is not much different from `ApplicativeError`/`MonadError` handling.
However, if there is a class, which has two `ApplicativeError` context bounds, the compiler will fail because of two implicit `Applicative` instances.
Then it's possible to extract these implicit in a separate argument list but there is a better alternative.
Alternatively, we can use `Handle` from Tofu.  
We have defined the `Handling` type alias for `DepartureError` handling and we have exactly the same alias for `ArrivalError`.
Station routes require both and it is not a problem, as they don't rely on `Applicative`, `Monad` and friends:
```scala
class StationRoutes[I[_]: Monad: DepartureError.Handling: ArrivalError.Handling]
```

In order to easily handle these errors, we need to import syntactic extensions again.
Note the difference, this time we are importing `handle` package: `import tofu.syntax.handle._`.
The simplified version of the code looks like this:
```scala
val registration = departures.register(newDeparture) *> Ok()

registration.handleWith(handleDepartureErrors)
```
Where `handleDepartureErrors` is just a function that converts a `DepartureError` to an HTTP response:
```scala
def handleDepartureErrors: DepartureError => I[Response[I]] = { 
  case DepartureError.UnexpectedDestination(city) => BadRequest(s"Unexpected city $city")
}
```
Similarly to `DepartureError` we create one for `ArrivalError`. 
The final code with error handling is available in [the repository](https://github.com/psisoyev/train-station-tofu/blob/master/route/src/main/scala/com/psisoyev/train/station/StationRoutes.scala).    

# Two effects for the price of one!
Have you ever considered having two effect types in your Tagless Final application?
The guys from Tofu did.
Let me show you why it could be a good idea.
And yeah, I've slightly exaggerated when I said about the price. 
It doesn't come for free, but you still might be interested.

Why would you need 2 different effects?
For those, who have some experience with `ZIO`, a situation when different services return different effects is totally normal.   
One service can require an environment that has some contextual data.
Others might not need it and shouldn't even know about the existence of the context.
Of course, context can be passed as a parameter, but then the service interface is littered with unnecessary data.
Also, there can be two different effects for initialization and runtime.
For example, the initialization effect, which starts all the services and creates resources can be `IO` monad.

Tofu provides a way to clean up your interfaces from a context.

First of all, nothing changes in the business logic services. Signatures are staying the same. There is only one effect.
However, this effect will be context-aware. We won't need to pass user or request information to method signatures.
We create our context at the very top, whenever a request comes into the system.
In the case of `train-station`, it is created in `StationRoutes` when the server receives a new request.
This means `StationRoutes` should be aware of both effects: 
```scala
class StationRoutes[
  I[_]: Monad: GenUUID: DepartureError.Handling: ArrivalError.Handling,
  F[_]: FlatMap: WithProvide[*[_], I, Context]
](...)
```
Here we have 2 effects: `I` which stands for initialization.
From the signature, we know that we will chain computations using this effect, generate new unique IDs, and handle business errors.
The second effect `F` is slightly more interesting. We know how to `flatMap` it, but also we know how to provide `Context` to run it.   
When we run this effect, it is converted to `I` and the result of it can be used later together with the first effect.

We will use this approach to provide tracing (`traceId`) and user (`userId`) information to business logic services.
First, let's have a look at the `Context` class we already used in the context-bound above:
```scala
case class Context(traceId: TraceId, userId: UserId)
```
`UserId` and `TraceId` are simple `newtype`s holding a `String` value.

For tracing and logging capabilities in the `Departures` service we build a `Context` and run it in routes: 
```scala
case req @ POST -> Root / "departure" =>
  for {
    departure <- req.asJsonDecode[Departure]
    userId    <- getUserId(req)
    traceId   <- I.randomUUID.map(id => TraceId(id.toString))
    register   = departures.register(action)
    res       <- runContext(register)(Context(traceId, userId))
  } yield res
```
After we have decoded a `Departure` from the request we need to fill the `Context` with the required fields.
We retrieve a `userId` from the incoming request
(in the repository code you will see that I've cheated and I'm just generating a random `userId` for every request) and generate a new tracing id.
Then we build `Context` and `register` the departure. That is it - we are running the effect with a specified context.
We will take a look at how we use the contextual information in the next chapter.
You can find the final implementation of the route [in the repository code](https://github.com/psisoyev/train-station-tofu/blob/master/route/src/main/scala/com/psisoyev/train/station/StationRoutes.scala#L34).

# Trace all the things
When running a modern high-throughput application it is important to have an ability to profile and monitor different parts of the application. 
As always there are several ways of doing it. In our train station simulator we will use tracing.
We won't use any specific tracing library but if you are looking for one, I would recommend having a look at [Trace4Cats](https://github.com/janstenpickle/trace4cats) or [Natchez](https://github.com/tpolecat/natchez).
We will create our own dummy tracing service, which will encapsulate the actual implementation.
It will have one simple method `traced`:
```scala
trait Tracing[F[_]] {
  def traced[A](opName: String)(fa: F[A]): F[A]
}
```

In our dummy implementation we will just log the trace message using `StructuredLogger` from `log4cats`:
```scala
def make[F[_]: FlatMap: StructuredLogger: WithCtx]: Tracing[F] = new Tracing[F] {
  def traced[A](opName: String)(fa: F[A]): F[A] =
    askF[F] { ctx: Context =>
      val context = Map("traceId" -> ctx.traceId.value, "operation" -> opName)
      F.trace(context)(s"[Tracing] $opName") *> fa
    }
}
```
As you can see, we just pretend to do tracing here. 
However, we are pretty serious with extracting context from our effect.
In the previous part we were talking about having a separate effect with a context. Now we have a chance to try it out.
Above, you can see a method `askF` which asks the `Context` from the effect.
We are able to use this method we have added `WithCtx` boundary, which is a type alias to `WithContext[F, Context]`. 
`WithContext` provides context information which can be explicitly requested.  

After getting the context we build our contextual `Map`, where we store `traceId` and operation name. 
This is logged and chained with the actual effect `fa`.

From the `Departures` service side it looks pretty simple:
```scala
private class Trace[F[_]: Tracing] extends Departures[Mid[F, *]] {
  def register(departure: Departure): Mid[F, Departed] = _.traced("train departure: register")
}
```
We have a `Mid` called `Trace`, which requires `Tracing` class, which we've implemented above. 
For the better user experience we've implemented `traced` extension method in `Tracing` companion object, 
that allows us to call `traced` method straight on the effect. 
See the full `Tracing` code [here](https://github.com/psisoyev/train-station-tofu/blob/master/service/src/main/scala/com/psisoyev/train/station/Tracing.scala).

So now, by using a simple context reader and `Mid` we have cleaned up our main logic from tracing utilities. 
All the code is fully decoupled and can be easily edited, replaced or even removed.

# Context aware logging
# JSON formatted logs

# Summary
Thank you for reading up to this point. 

Let me know if you have any questions or suggestions in the comment section! Feedback is also very welcome, thank you!
Cheers!