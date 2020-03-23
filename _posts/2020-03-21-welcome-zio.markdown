---
layout: post
title: Implement your future with ZIO [RC18 edition]
date: 2020-03-21 13:37:00 +0100
description: This post will help you to start building Scala applications with ZIO
img: welcome-zio/my-pager.png # Add image post (optional)
tags: [Scala, ZIO, ZLayer, http4s]
comments: true
---

This post will help you start building Scala applications with ZIO.

Today there are many libraries in Scala ecosystem, that promise to improve your efficiency. 
I wrote this post in order to help you to start with the new effect type on the block - [ZIO](https://zio.dev/). 
It's a huge library that gives you powerful tools to build concurrent applications and its own ecosystem.  

This post doesn't cover all the functionality but will help you get started with something bigger than a 'Hello World' app.
On certain topics I will not go into details, but rather provide links to allow you to do your own research. 
The intent of this post is to familiarize you with the library on a high level. 
In the next chapters I would like to cover more specific parts of ZIO ecosystem and guide you in a deep dive into the different parts of the library.

Please note that we will be using ZIO version `1.0.0-RC18-2`. 
We should not expect to see ZIO API changes in the next release candidates before the official release of version 1.0. 
If for some reason you are interested in ZIO version `1.0.0-RC17` you are welcome to read [first edition of this post](https://scala.monster/welcome-zio-old/).

If you prefer reading code rather than text you please check the [project page](https://github.com/psisoyev/release-pager). 

## The problem to solve

##### We look for tools that solve our problems, not problems to solve with our beloved tools. Otherwise, we end up with a zoo of different technologies that is not sustainable.  

In [the previous article]({{site.baseurl}}/design-a-pager/) we have defined a problem that needs to be solved. 
If you haven't read it yet, I would recommend checking it before you proceed with this one.
 
Scala ecosystem is diverse and can provide you with various different approaches and techniques that can solve the very same business problem. 
There are different frameworks and approaches to build applications. 
ZIO is one of the newest libraries in the ecosystem and is advertised that it can simplify software development with Scala, which would make its users more efficient.
As I like to explore the world of functional programming I decided to try it out and share my experience with you.  

According to [documentation](https://zio.dev/docs/overview/overview_index) ZIO is a library for asynchronous and concurrent programming that promotes pure functional programming.
If functional programming is something you like or you feel interested in it, this should spark some curiosity in you. 

Let's take a look at what ZIO is about.
ZIO allows you to build your programs in a "lazy" fashion. You describe how your program should behave in [pure functions](https://en.wikipedia.org/wiki/Pure_function).
These functions return data structures that are called functional effects. In short, a functional effect is an immutable value that describes tasks that should be done. 
When we create such an effect we have to run it manually. This means that all side-effects inside of it will be evaluated only after interpreting. 
You combine these effects into a program, which you run only once, on the very top level. 
If you are not familiar with this concept I would recommend you to watch [this presentation](https://youtu.be/30q6BkBv5MY).
 
Functional effects make unit testing of side-effects quite easy. This will be covered in detail in the next article. 
For now, it's important to mention, that ZIO has its own testing framework to test functional effects.

ZIO has tools to build low-level concurrency constructs. The library has its own implementation of [fibers](https://en.wikipedia.org/wiki/Fiber_(computer_science)). 
Fibers can be treated as lightweight "green threads". They have low memory overhead, dynamic stacks and are garbage collected when they are not needed anymore.
There is a set of high-level operations built on top of fibers and it is recommended to use these operations when writing high-level business code. 

The library has resource management features that provide strong guarantees of finalization and clean up. 
Even if the processing of a resource data fails, ZIO guarantees to perform resource clean up to avoid memory leaks. 
It is similar to `try/catch` block but expressed as a functional effect.
  
Also, ZIO has its own implementation of software transactional memory, streaming and other higher level concurrency constructs. 
There are many various projects in the [ecosystem](https://github.com/zio/) that you might be interested in. 
There is even an [actor model](https://github.com/zio/zio-actors) implementation.
Some of them are in the development phase and if you ever wanted to participate in open source community that is your chance.  

## The solution

### Getting started with ZIO

If you already have heard about ZIO you know that at its core lays a data type which describes an effect.
A value of this type doesn't do anything itself. It is just a description of something that should be done.
ZIO runtime system is responsible for actually doing what is described or simply - interpreting. 
Usually it is done only once in our `run` function in `Main` class. 
`ZIO[-R, +E, +A]` data type has 3 type parameters:
* R - type of environment required to interpret the effect
* E - error type
* A - return type

You can see it as "E or A can be produced when R is provided". 
If we would speak functions that would be `R => Either[E, A]`. 
If you want to learn more about IO monad refer to the [official documentation](https://zio.dev/docs/datatypes/datatypes_io). 
Now let's see how can we use it in service implementation.

In [the previous article]({{site.baseurl}}/design-a-pager/) we have defined logical services. It is time to implement them.    

To structure the application services we will use [module pattern](https://zio.dev/docs/howto/howto_use_layers).
Every module will be expressed as an object. Inside of this object, we have a service definition, which we will have to implement.
As always, it is recommended to use descriptive names for the module definition. 

Dependency injection in ZIO is done using layers. You can see layer as a recipe to cook a service. 
Type signature of `ZLayer` is very similar to `ZIO` data type - `ZLayer[-RIn, +E, +ROut <: Has[_]]`.
Ok, this looks complicated but in a fact it is not:
* RIn - required dependencies to build this layer (can be `Any` in case if there are no dependencies)
* E - possible error type (can be `Nothing` in case if there are no errors expected)
* ROut - type of the layer we want to cook. We expect our service type to be inside of a `Has` type. 
`Has` is a data structure that holds a heterogeneous map with a mapping from service type tag to service implementation. 
 
Let's refresh our memory and see how the service diagram of the application could look like.

![Service diagram]({{site.baseurl}}/assets/img/design-a-pager/services.png#center)

As you could see in the service diagram the heart of the application is subscription service. 
It should be able to store user subscriptions to GitHub repositories and latest repository versions. 
Let's define the subscription logic module.
It is possible to use both Scala `object` and `package object` for module definition. 
Here we have defined `SubscriptionLogic` module, which has subscription service definition:

```scala
object SubscriptionLogic {
  type SubscriptionLogic = Has[Service]

  trait Service {
    def subscribe(chatId: ChatId, name: Name): Task[Unit]
    def unsubscribe(chatId: ChatId, name: Name): Task[Unit]

    def listSubscriptions(chatId: ChatId): Task[Set[Name]]
    def listRepositories: Task[Map[Name, Option[Version]]]
    def listSubscribers(name: Name): Task[Set[ChatId]]

    def updateVersions(updatedVersions: Map[Name, Version]): Task[Unit]
  }
}
``` 

Above you can see the definition of the subscription service interface. 
Here we have defined several actions that this service can handle.
Methods return us `zio.Task` type. This is a type alias to `ZIO[Any, Throwable, A]`. 
In this case, the environment type is not used, but there might be an exception thrown by the DB layer (eg. lost DB connection).
Usually, we would catch expected errors and wrap them into a typed error. Here, to keep things simple, lets use `Throwable`.

To implement the logic we have to create a class which extends `SubscriptionLogic` trait. 
There are two ways to organize your implementations: either put all implementations inside of the module or create a separate file in the same package.
The difference is, that if you will have a service with several implementations, it won't be convenient to navigate in several thousands of lines of code in one file.
Let's create our first implementation of a module:

```scala
private[subscription] final case class Live(
  logger: Logger.Service,
  chatStorage: ChatStorage.Service,
  repositoryVersionStorage: RepositoryVersionStorage.Service
) extends SubscriptionLogic.Service {
  override def subscribe(chatId: ChatId, name: Name): Task[Unit] =
    logger.info(s"$chatId subscribed to $name") *>
      chatStorage.subscribe(chatId, name) *>
      repositoryVersionStorage.addRepository(name)

  override def unsubscribe(chatId: ChatId, name: Name): Task[Unit] =
    logger.info(s"Chat $chatId unsubscribed from $name") *>
      chatStorage.unsubscribe(chatId, name)

  ... // skipped the rest
``` 

Lets skip the rest of the implementation in this snippet, you should get the idea. 
You can find the full version of it on [GitHub](https://github.com/psisoyev/release-pager/blob/master/service/src/main/scala/io/pager/subscription/Live.scala). 
`Live` has three dependencies: a logger, chat storage and repository version storage.
Other implementation of this logic might have a totally different set of dependencies or even have no dependencies at all.
Here we see that it is just a class, that accepts some services as constructor arguments. 
It shouldn’t be anything new for us.

In the code above we see the implementation of two functions, which log user action and call chat storage.
For those, who find function aliases unreadable or not familiar with them, `*>` or as I call it "ice cream", is just an alias to `flatMap` function, which drops the result of the previous computation.   
Here subscription logic doesn't have any clue how the storage will work and it should not change. 
Note that I use the word `storage` for services, that are responsible for saving our precious data. 
This name is quite abstract and doesn't imply any implementation details.
As an example let's see what is inside of `ChatStorage` module. 
I have created two versions of the storage and one of them is in-memory. Take a look:   

```scala
object ChatStorage {
  type ChatStorage = Has[Service]                           // 1

  trait Service {
    def subscribe(chatId: ChatId, name: Name): Task[Unit]   // 2
    def unsubscribe(chatId: ChatId, name: Name): Task[Unit] // 3

    def listSubscriptions(chatId: ChatId): Task[Set[Name]]  // 4
    def listSubscribers(name: Name): Task[Set[ChatId]]      // 5
  }
}
```

This is the definition of our storage logic:
1. a type alias to describe a layer dependency (will be used later in layer construction)
2. add a new subscriber to a repository
3. remove a subscriber from a repository
4. list all the subscriptions for a specific chat
5. list all the subscribers for a specific repository

As we saw before there might be several service implementations - we will have in-memory and SQL database implementations. 
In the scope of this article, we will use only the in-memory implementation. 
For the in-memory implementation we will use `Ref` data type from ZIO. You can read about it [here](https://zio.dev/docs/datatypes/datatypes_ref). 
In short, `Ref` is a mutable reference to a value, which in this case is an immutable `Map` that stores all chat subscriptions (repository names).
ZIO takes care of the concurrent operations on `Ref` and guarantees the atomicity of all operations on the `Map`. 
Here the requirements from the service are quite low - to be able to read the current state and update it when the user is changing his subscription list. Concurrently, of course.
We create a new file besides the module definition object:

```scala
private[chat] final case class InMemory(subscriptions: Ref[SubscriptionMap]) extends Service {
  type RepositoryUpdate = Set[Name] => Set[Name]

  override def subscribe(chatId: ChatId, name: Name): UIO[Unit] =
    updateSubscriptions(chatId)(_ + name).unit

  override def unsubscribe(chatId: ChatId, name: Name): UIO[Unit] =
    updateSubscriptions(chatId)(_ - name).unit

  private def updateSubscriptions(chatId: ChatId)(f: RepositoryUpdate): UIO[Unit] =
    subscriptions.update { current =>
      val subscriptions = current.getOrElse(chatId, Set.empty)
      current + (chatId -> f(subscriptions))
    }.unit

  override def listSubscriptions(chatId: ChatId): UIO[Set[Name]] =
    subscriptions
      .get
      .map(_.getOrElse(chatId, Set.empty))

  override def listSubscribers(name: Name): UIO[Set[ChatId]] =
    subscriptions
      .get
      .map(_.collect { case (chatId, repos) if repos.contains(name) => chatId }.toSet)
}
``` 

A lot is happening in this small code snippet. 
As you might have noticed, the return type of all the methods is not `Task` as in the interface, but `UIO`.
`UIO` is used when you are sure that all the operations are pure and nothing can break. 
Basically, that there are no errors expected (or even possible). 
ZIO can guarantee that operations on `Ref` are pure, but it has no control over your actions. 
If you want to throw an exception inside of the update function - you can do it, the compiler does allow that. Should you do it? Never.
Writing code with functional effects requires attention and discipline. Also, it's always great to have a code review afterwards.
The difference between `Task` and `UIO` is that error type of `UIO` is `Nothing` instead of `Throwable`. 
Having this we can expect that this function will not fail.

Updating the value inside `Ref` is simple. Just call `update`, take the provided state and change it as you wish.
That is why we have defined `updateSubscriptions` function, which gets the current value of `Ref`, finds user subscriptions and updates them with provided function.
So far we have only two functions - add and remove a subscription. The difference is in one sign: `+` in case of addition and `-` in case of deletion.        

Reading the value is even simpler - you just need to call `get` on `Ref` and you have the current state. 
Code looks clean and concise.

What if you have decided to implement a SQL table, which will store same data? You just create another implementation, for example `Doobie`:

```scala
private[chat] final case class Doobie(xa: Transactor[Task]) extends Service {
  override def subscribe(chatId: ChatId, name: Name): Task[Unit] = ???
  override def unsubscribe(chatId: ChatId, name: Name): Task[Unit] = ???
  override def listSubscriptions(chatId: ChatId): Task[Set[Name]] = ??? 
  override def listSubscribers(name: Name): Task[Set[ChatId]] = ???
}
```   

[Actual implementation](https://github.com/psisoyev/release-pager/blob/master/storage/src/main/scala/io/pager/subscription/chat/Doobie.scala) is not important at this point, but you can see that now there is a different set of dependencies comparing to `InMemory` implementation. 
We don't need to have `Ref` as we will store the data in a database. The only dependency for this implementation is `doobie.util.transactor.Transactor`. 

Now, if we want to replace `InMemory` implementation with `Doobie` implementation we have to change only one place - the `Main` class where all the services are wired together. 
As we design our services against interfaces we don't care about implementation details of the dependee, so `SubscriptionLogic` implementations won't be affected by the change. 

Now we should have some basic understanding of how to build a business logic service with ZIO. 

### Wiring up
If you read so far, you've seen several service definitions and hopefully, you have a high level picture of what we are doing here.
At this point, we could go through the rest of the service definition and implementation, but it's not very different from what we've already seen except some small details.
It would be interesting to see how we are starting the application and connect dependent services.

As we are building the application from scratch we will extend our `Main` object with `zio.App`.
Overriding the `App` forces us to override `run` method:

```scala
override def run(args: List[String]): ZIO[ZEnv, Nothing, Int]
```

As you can see, the program expects us to return a ZIO effect, which the library will run. 
The first argument - `ZEnv` is default ZIO environment, that will provide us:
* `Clock` - access to system time, sleep function
* `Console` - access to the console (printing, reading input)
* `System` - access to environment variables
* `Random` - access to the randomizer
* `Blocking` - access to blocking thread pool

From this set of built-in services we will use: 
* `Clock` - to schedule GitHub repository latest version retrieval
* `Console` - to log messages in stdout
* `System` - to read Telegram token which is stored in an environment variable

Let's take a look at how the whole program assembly looks like:
```scala
  val program = for {
    token <- telegramBotToken

    http4sClient <- makeHttpClient
    canoeClient  <- makeCanoeClient(token)

    _ <- makeProgram(http4sClient, canoeClient)
  } yield ()
```

Here we prepare all the necessary inputs to build the program. As the first step, we retrieve Telegram bot token from an environment variable.
Then we create `Http` and `Telegram` clients. After that we wire all services together in `makeProgram` method.

ZIO gives you the ability to describe method calls on a service without having an instance of the service. 
This is where `R` type parameter in ZIO data type comes in handy.
We can see it as taking a service in debt. Of course, we must then return the debt.
How that works?
Let's take a look at the program itself:
```scala

val startTelegramClient: ZIO[TelegramClient, Throwable, Unit] = 
  ZIO.accessM[TelegramClient.Service](_.start).fork

val scheduleReleaseChecker: ZIO[ReleaseChecker, Throwable, Unit] =
  ZIO
    .accessM[ReleaseChecker.Service](_.scheduleRefresh)
    .repeat(Schedule.spaced(1.minute))

val program: ZIO[Clock with ReleaseChecker with TelegramClient, Throwable, Int] = 
  startTelegramClient *> scheduleReleaseChecker
```  

First, we access a `TelegramClient` service, that will be provided later, call `start` method on it and fork into a separate `Fiber`. 
As it was mentioned above, `Fiber` details are out of the scope of this article.
For now, to keep things simple, imagine that we have spawned a separate thread for it (it is not exactly correct, but you should get the point).

Then we access a `ReleaseChecker` service. 
We call a `scheduleRefresh` method, which will call GitHub API and will check for new repository versions.
To repeat the refresh effect every minute we call `repeat` method on ZIO data type. 
If at some point of time `scheduleRefresh` will fail with an error, repeating of the effect will stop.

We combine both effects using the "ice cream" (flatMap) method and now we have one program, with environment `Clock with ReleaseChecker with TelegramClient`.
`Clock` appears as a required service because we are using `repeat` method.
The very last thing is to provide these missing services to the library and we are done. 
We have to fulfill all the dependency requirements of the services we defined before.
For example, `Live` implementation of `ReleaseChecker` depends on `SubscriptionLogic` and the compiler expects this service to be provided.

It is possible that the same method could be called using `accessM` in different parts of the application. 
To avoid spreading of boilerplate across the application we can create a convenience methods inside of a service module:
```scala
object ReleaseChecker {
    ... // skipped the rest
  def scheduleRefresh: ZIO[ReleaseChecker, Throwable, Unit] = ZIO.accessM(_.get.scheduleRefresh)
}
```

As we learned before, ZIO uses `ZLayer` to provide necessary dependencies. 
Layers between themselves can be merged (or mixed in) and provided together.

Let's see how we [define](https://github.com/psisoyev/release-pager/blob/master/service/src/main/scala/io/pager/subscription/SubscriptionLogic.scala#L28) the recipe to build `Live` implementation of `SubscriptionLogic`: 
```scala
  def live: ZLayer[Logger with ChatStorage with RepositoryVersionStorage, Nothing, Has[Service]] =
    ZLayer.fromServices[Logger.Service, ChatStorage.Service, RepositoryVersionStorage.Service, Service] {
      (logger, chatStorage, repositoryVersionStorage) => 
        Live(logger, chatStorage, repositoryVersionStorage)
    }
```

To build this implementation of the service we need 3 other services: `Logger`, `ChatStorage` and `RepositoryVersionStorage`.
Note, that these services are dependency type aliases we have defined in modules (remember `type ChatStorage = Has[Service]`?).
There are several possibilities to build a ZLayer. 
Here we are using `fromServices` method which expects dependencies to be defined in type parameters together with the resulting type (the `Service` in the end).
Inside of this function we create an instance of the service.
Another way of building a ZLayer is using an effect or a managed resource. 
When using these it is more likely that one of them can fail.
When we are building an `inMemory` storage we have to provide a `Map` inside of a `Ref`. 
Creation of a ref is an effect. To create a layer for it we will use `fromEffect` method: 
```scala
val versionMap = ZLayer.fromEffect(Ref.make(Map.empty[Name, Option[Version]]))
val subscriptionMap = ZLayer.fromEffect(Ref.make(Map.empty[ChatId, Set[Name]]))

val logger = Logger.console

val chatStorage = subscriptionMap >>> ChatStorage.inMemory
val repositoryVersionStorage = versionMap >>> RepositoryVersionStorage.inMemory
val storage = chatStorage ++ repositoryVersionStorage
val subscriptionLogic = (logger ++ storage) >>> SubscriptionLogic.live
``` 

Yes, we have your "favorite" symbolic aliases here. 
However, they are pretty intuitive. 
`++` will combine layers horizontally and `>>>` vertically. 
Let's have a look at what we are doing above and you will get it.

To create `chatStorage` layer we provide `versionMap` to (`>>>`) `ChatStorage.inMemory` recipe using vertical composition.
This means that vertical composition takes output of one layer and puts it into input of another one.

When we have both storage layers created we can combine them horizontally with `++` to create `storage` layer.
This means that horizontal composition combines layer outputs.

We combine `storage` with `logger` horizontally and provide it to `SubscriptionLogic.live` layer.
That is it. 
Now we have `subscriptionLogic` which is of type `ZLayer[Clock with Console, Nothing, Has[SubscriptionLogic.Service]]`. 
Let's take a closer look on the type parameters of the layer:
* Clock with Console - we expect these services to be provided by the framework
* Nothing - we do not expect errors when we are creating this layer
* Has[SubscriptionLogic.Service] - result of this layer recipe, the service itself

You can see the whole `Main` class [here](https://github.com/psisoyev/release-pager/blob/master/backend/src/main/scala/io/pager/Main.scala).   

## Summary
I'm happy if you have read until this point.
As this is really a high-level introduction to ZIO capabilities this article doesn't provide you with a lot of details and also it doesn't compare ZIO with other solutions.
 
I have introduced you to ZIO. You are not close friends yet, but we'll get there eventually.
We have seen how to design and implement several services. We have seen how to build dependencies between these services. 
Also, we've used ZIO environment and ZLayer to create service instances and start our program. 
Without noticing we were actively using Fibers, which is the smallest concurrency element in ZIO.

After becoming familiar with module pattern and few concepts it is becoming easy to reason about the code.
It looks nice and clean. 

Would I recommend ZIO? It depends on what you are trying to build. 
If you want to learn something new or want to be aware of the trends in Scala world - try it without any doubts. 
If you are building your own multi-billion startup which won't go live next Tuesday I might go for it. 
If you are building a general-purpose library, which would not be a part of ZIO ecosystem? Emm, could be. Why I'm not so sure? 
There are still people who are afraid of 'Z' in the library names. 
Also, users of the library will have ZIO in their dependency tree which might not be the desirable solution. 
There are pros and cons to use Tagless Final style for this purpose. Such a comparison deserves a separate article. 
Of course, you can use ZIO in your Tagless Final applications as the effect type.

If your team is not very proficient with trendy functional programming terms like ~~EJB, inheritance~~ "effect", "Tagless Final", it might be challenging. 
That, of course, depends on people, their ability and will to learn, project requirements and deadlines.
However, for me it feels that getting started with ZIO might be quite an easy thing to do.
To start writing some ZIO code you do not have to be proficient with category theory terminology, for example.

Using this kind of "God monad" is dangerous. 
If you won't understand how the framework works you might end up catching weird bugs which won't be bugs but misuse of features.
With ZIO it is easy to write code that will work somehow magically, but I think that in longterm it is important to understand how it is working.
Of course, that applies to any new technology.

In general, functional programming requires attention, discipline and understanding of the things you do (that applies to any kind of programming).
If you are familiar with Cats Effect, then ZIO shouldn't be hard for you. 
Some concepts and techniques are similar, just some names might differ.

ZIO provides a lot of convenience methods, e.g. `fork` and `repeat` that we saw before. 
These methods are quite useful and make your code easier to read and reduce amounts of boilerplate. 
However, you have to get used to them. 
With symbolic aliases it is a bit more challenging. Even though using them is not mandatory and you can code without them.
Code of the library is well organised and it is easy to navigate through its files. Also, it has a lot of ScalaDocs.

It is always easy to get some help in [ZIO discord chat](https://discord.gg/2ccFBr4). 
Community is already quite big and it is rapidly growing. 
There are many side projects around ZIO including its own implementation of [actors](https://github.com/zio/zio-actors), [Redis client](https://github.com/zio/zio-redis), [Kafka client](https://github.com/zio/zio-kafka) etc. 

I will continue exploring ZIO and in the next article I will share my experience with unit testing possibilities in ZIO.
Follow me on [Twitter](https://twitter.com/scalamonster) if you would like to be notified about a new article.