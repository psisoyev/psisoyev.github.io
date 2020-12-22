---
layout: post
title: Upgrade your Tagless Final with Tofu
date: 2020-12-18 13:37:00 +0100
description: This post introduces Tofu toolkit to Scala developers
img: train-station/trains.png
tags: [scala, apache pulsar, neutron, zio, tagless final, fs2, tofu]
comments: true
hidden: true
---

In my previous article I've created an event-driven system built on top of Apache Pulsar. 
The system was built in Scala and was using Tagless Final technique. 
In this article I would like to introduce you to a cool kid on the block that you might not know yet. 
Meet [Tofu](https://github.com/TinkoffCreditSystems/tofu) - a functional programming toolkit aimed at taming the complexity of Tagless Final approach.
I will show how you can use utilities from Tofu to improve your codebase. 
However, I must warn you that some of the concepts might be advanced and look complex at the first sight but are actually pretty simple when you have a closer look.

In order to compare code improved with Tofu with traditional Tagless Final application I've cloned the [train-station](https://github.com/psisoyev/train-station) repository.
We will use the same application to improve it with Tofu and then we will be able to compare the results.
If you are like me and prefer reading code first and then read the article jump straight to the [new repo](https://github.com/psisoyev/train-station-tofu).

In this article I will show you how to:
* clean up your business logic from boilerplate; 
* improve error handling;
* split runtime effect from application initialization effect;
* enable context aware tracing;
* enable context aware logging;
* enable JSON formatted logging;
* ???

I can't say that you must use all of that in your application as you and your team can already have built your own habits and style. 
My personal feeling is that Tagless Final is a great way of describing your application, however, sometimes the current ecosystem lacks of tooling. 
The idea of this post is to show you what is possible but it doesn't mean it will solve all your problems.
You can use only part of the toolkit to solve particular problems. For that, the toolkit is split into several sub-modules. 
More info on this you can find in the [README file](https://github.com/TinkoffCreditSystems/tofu#quick-start).

The toolkit has much more to offer you: 
* Agent - `Ref` on steroids, which allows effectful updates;
* Granular typeclasses for forking and racing;
* Env - a monad which allows composition of functions that are context-aware;
* Optics;
* Many other small but very useful utilities.

# Raki na mide ???
The first thing we will see today is how to clean up application business logic from cross-cutting concerns surrounding the core of the logic. 
For this we will use a class called `Mid`. 
It might slightly remind you aspect-oriented programming.  
The idea is very simple - we extract utility logic from the core logic. 
For example, logging or input data validation can be extracted into small separate modules. 

We will start with cleaning up `Departures` service which we have created in the [previous article](https://scala.monster/train-station/).
If you haven't read the article or already have forgotten what it is about, you always can [find the service implementation on GitHub](https://github.com/psisoyev/train-station/blob/master/service/src/main/scala/com/psisoyev/train/station/departure/Departures.scala#L31).
I can see at least 2 things we could separate from the core logic. 
The first one is logging before and after the method call. 
The second one is input data validation. 

Let's first agree on what is the core implementation logic of departure registration. 
In our case it is generation of a random uuid and creation of the event, which we return as the result of registration.
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
Application can work only having this and the rest is not important. 
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
If you have read the code carefully you could notice that now we extend `Departures` trait not with just the effect type as we did with the core logic, but with `Mid` -
`extends Departures[Mid[F, *]]`. Also, the return type of registration is not a simple event wrapped in `F`, but is a `Mid`. 
As for a casual user we won't even notice a difference. 
Again we have to override `register` method. This time as we use `Mid`, we receive a new input - result of the `registration` of type `F[Departed]`.
We basically surround the resulting effect with two other effects - a log before and a log after the registration.
The ice-cream symbols `*>` and `<*` are just symbolic aliases to `flatMap` method which drops the output.
Aand that's it. We don't need anything else here for logging. 

If you think this logging class was simple, then validation will be even simpler for you.
We create a new class:
```scala
class Validate[F[_]: Monad: Raise[*[_], DepartureError]](connectedTo: List[City]) extends Departures[Mid[F, *]] {
  def register(departure: Departure): Mid[F, Departed] = { registration =>
    val destination = departure.to.city

    connectedTo
      .find(_ === destination)
      .orRaise(UnexpectedDestination(destination)) *> registration
  }
}
```
Here we add a new context bound which we haven't seen before - `Raise`. 
We will have a closer look at it later, when we will be talking about error management.
All the rest looks similar to logging class we saw before - we extends `Departures` trait with `Mid` as the effect. 
However, this time, the only thing we need is to do some actions before calling the core logic. 
So we run our validation logic and call `registration` method `orRaise` an error. 

The only missing thing is gluing all the pieces together. 
e will use the same `make` method we had before, where we will initialize the classes and wire everything together. 
```scala
def make[F[_]: Monad: GenUUID: Logger: Raise[*[_], DepartureError]](
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

Wait, this doesn't compile! We are missing the one last thing - we can't `combine` instances without an implicit `ApplyK` in the scope.
This sounds complicated and might scare some people away..
Luckily, the easiest way to get an instance of it is to simply add an annotation on the original interface:
```scala
@derive(applyK)
trait Departures[F[_]] {
```
This `derive` annotation comes from another cool library called [`derevo`](https://github.com/manatki/derevo). 
The purpose of this library is different instance derivation using a single macro annotation.

Now the code compiles and we can be happy about having cleaner business logic. 
Here is the [link to the final code](https://github.com/psisoyev/train-station-tofu/blob/master/service/src/main/scala/com/psisoyev/train/station/departure/Departures.scala#L24).
Yes, the final version has slightly more lines of code, but it's a low price to pay for the clean code.

You might be not convinced with the example above because in your case you don't log inputs and outputs, validation service is a separate entity.
We could do this with `Arrivals` as it's validation is much bigger and potentially could grow even more. 
At the same time, we could use the same `Mid` concept even in the validator itself. [Have a look](https://github.com/psisoyev/train-station-tofu/blob/master/service/src/main/scala/com/psisoyev/train/station/arrival/ArrivalValidator.scala#L20).
There are more ideas what you could extract to `Mid`:
* tracing;
* authentication;
* authorisation;
* monitoring;
* caching;
* persistence;
* event publishing;
* and even more.

Another bonus of having core logic extracted is that you can substitute the logic itself without changing the plumbing around.
It means that logging, monitoring, tracing and all the other "not so important" stuff will stay as is.


# Summary
Thank you for reading up to this point. 

Let me know if you have any questions or suggestions in the comment section! Feedback is also very welcome, thank you!
Cheers!