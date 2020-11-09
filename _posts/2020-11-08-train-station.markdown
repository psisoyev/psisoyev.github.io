---
layout: post
title: Event-driven railway network based on Pulsar
date: 2020-11-08 13:37:00 +0100
description: This post introduces Apache Pulsar to Scala developers
img: train-station/trains.png
tags: [scala, apache pulsar, neutron, zio, tagless final, fs2]
comments: true
hidden: true
---

I took this photo while crossing [Landwasser viaduct](https://en.wikipedia.org/wiki/Landwasser_Viaduct) in Filisur, Switzerland.
Switzerland is famous for [its railway network](https://en.wikipedia.org/wiki/Rail_transport_in_Switzerland). 
According to Wikipedia, it is the densest railway network in the world.
How about creating a virtual simulator of it? Sounds great!

In this article, I would like to introduce you to [Apache Pulsar](https://pulsar.apache.org/) and [Neutron](https://github.com/cr-org/neutron).
Apache Pulsar is an open-source distributed pub-sub messaging system originally created at Yahoo and now is part of the Apache Software Foundation. 
Often it is compared to Apache Kafka.
If you are interested in a comparison of these two systems you can find several articles on this topic. 

Also, we will use Neutron. It is a Pulsar client which is based on [FS2](https://fs2.io/) streaming library. 
Be aware, even though Neutron is developed and used in production by [Chatroulette](https://about.chatroulette.com/), it is still in active development.

In my childhood, I always wanted to have a huge toy railway but never had one. 
Now I can build a virtual simulator on my own.  
In this tutorial, we will build an event-driven railway network simulator together.

# The idea
We will build a railway network MVP consisting of three stations: Geneva, Bern, and Zurich. 
Geneva and Zurich are connected to Bern and are not connected to each other.

![City connection]({{site.baseurl}}/assets/img/train-station/cities.png#center)

Every station will be represented as a node. 
Connected nodes will communicate through a message broker - Apache Pulsar.
Every node consumes events that are emitted by connected nodes. 
Listeners (consumers) will filter incoming events and will use the ones which relate to a specific city.

We should have a way to control the behavior of the simulator. 
One way of doing this is adding HTTP endpoints, which can be used for manual intervention. 
By sending an HTTP request, the user will be able to add new trains to the system. 

In this tutorial we won't persist any data, so we won't use any database or cache system. 
All the data will be kept in memory. 
For that, we can use a high-level concurrency mechanism like [`Ref`](https://zio.dev/docs/datatypes/datatypes_ref).

At the core of our system will be Apache Pulsar. 
It will be responsible for communication between nodes.
After every change in its state system should emit a new event. 
The event would describe an action that already has happened.
This means every event should have a `timestamp`. 
Also, every event should have a `trainId`, which denotes an identification number of a specific train.
At the very beginning there will be two events: 
- `Departed` - is emitted when a train has departed from a city. 
- `Arrived` - is emitted when a train has arrived in a city.
Both events contain generic train information: its identification number, city of departure, destination city,
expected time of arrival and event timestamp.  

Every city consumes events from connected cities. For example, Zurich consumes Bern events and doesn't know about events coming from Geneva.
Event consumer in Zurich should make sure that whenever Bern emits a `Departed` event with the destination city set as Zurich it should be captured.
Every city will have its own topic. In the future, if we will need to optimize this we can split generic "city topic" into a few, more specific topics.
For 3 cities there will be 3 topics.  

As a glue between our business logic and Apache Pulsar we will use [neutron](https://github.com/cr-org/neutron).  
Every consumed topic will be converted to a `fs2` stream which we already know how to handle. 
If you don't, I really recommend going through [fs2 guide](https://fs2.io/guide.html), but it is not mandatory to understand the code below.

The application will be written using the Tagless Final technique based on `cats` library and using `ZIO` as the runtime effect. 
This is a controversial decision and I have different thoughts about it but it deserves a separate blog post.

# Apache Pulsar in ~10 sentences
Apache Pulsar is a distributed messaging and streaming platform. 
It can be used to build a highly scalable system on top of it. 
Parts of a system(s) can communicate using messages in up to millions of topics.
From a developer perspective, it can be treated as a black box, but I would recommend learning more on how it works under the hood.

To understand this article you will need to be familiar with a few concepts (simplified):
* `Topic` - channel for transmitting messages. There are 2 types:
  * `Persistent` - message data is persistently stored;
  * `NonPersistent` - message data is never persistently stored and is kept in memory. All in-transit messages will be lost if a Pulsar broker will go down;
* `Producer` - process that attaches to a topic and publishes messages;
* `Consumer` - process that attaches to a topic via a subscription and then receives messages;
* `Subscription` - configuration rule that determines how messages are delivered to consumers. There are 4 subscription types:
  * `Exclusive` - Single consumer. Every other consumer of a subscription in this mode will raise an error; 
  * `Failover` - Multiple consumers can be attached in this mode, but only one will receive messages;
  * `Shared` - Multiple consumers will receive messages in a round-robin distribution;
  * `Key Shared` - Multiple consumers will receive messages distributed by key (one key will always be delivered to the same consumer).

Imagine a system, 
that is emitting events, 
which are processed by a `Producer`, 
that publishes them to a `Topic`, 
which is listened to by a `Consumer` of another system, 
that is attached using a `Subscription`.

Simple as that. Almost. In a fact, it is a slightly more complicated process 
and if you want to learn more about Apache Pulsar [check out its documentation](https://pulsar.apache.org/docs/en/pulsar-2.0/).

# The multi-billion business logic 
Earlier we mentioned two events that can happen on the railway - train departure and arrival. It is time to define them:
```scala
case class Departed(id: EventId, trainId: TrainId, from: From, to: To, expected: Expected, created: Timestamp) extends Event
case class Arrived(id: EventId, trainId: TrainId, from: From, to: To, expected: Expected, created: Timestamp)  extends Event
```
These events provide general information about an action that already happened in the system: unique event id, train id, departure city, 
destination city, expected arrival time, and actual event timestamp. 
Later we can add additional information such as platform number, number of cars, etc.  
For the sake of simplicity of the tutorial let's limit the amount of data we need for the system to work.
Every field in the event is strongly typed as we don't want to accidentally mix up things (for example, destination and departure cities). 

As we don't have an automatic system that would detect train arrival or departure, we will have to control our network manually. 
We could imagine a real person (a train dispatcher) who controls the network. 
This person would press buttons on a shiny panel full of buttons and lights.
We won't have a shiny UI, but we can build an API for it. 
At the core of this API would be 2 simple commands:
```scala
case class Arrival(trainId: TrainId, time: Actual)
case class Departure(id: TrainId, to: To, time: Expected, actual: Actual)
```
These commands would trigger the business logic of a train station.

## Train departure
We start by creating a train departure. 
This is the first command that we should be able to send using a simple cURL:
```
curl --request POST \
  --url http://localhost:8081/departure \
  --header 'content-type: application/json' \
  --data '{
	"id": "153",
	"to": "Zurich",
	"time": "2020-12-03T10:15:30.00Z",
	"actual": "2020-12-03T10:15:30.00Z"
}'
```
This command is assuming Bern service node is running on port 8081. 
Every node is running an HTTP server and should be able to handle this request. 
As the HTTP server, we will use `Http4s` library. 
Our first route definition looks like this: 
```scala
case req @ POST -> Root / "departure" =>
  req
    .asJsonDecode[Departure]
    .flatMap(departures.register)
    .flatMap(_.fold(handleDepartureErrors, _ => Ok()))
```
Here we call `Departures` service, which we haven't defined yet. Let's do it now. 
The only thing service should do is to register a departing train:
```scala
trait Departures[F[_]] {
  def register(departure: Departure): F[Either[DepartureError, Departed]]
}
```
There are different ways to do data validation in Scala. 
I've picked the most straightforward and explicit way - returning an `Either` with a custom error. 
If registering the train succeeds we return a `Departed` event. 
If not, there is an error that should be handled by the caller. 

For simplicity reasons let's call the message producer inside the `Departures` service implementation. 
Wait, we didn't implement it yet? No need to wait, let's do it now.
Inside the `Departures` companion object we create function `make`:
```scala
object Departures {
  def make[F[_]: Monad: UUIDGen: Logger](
    city: City,
    producer: Producer[F, Event]
  ): Departures[F] = new Departures[F] {
    def register(departure: Departure): F[Either[DepartureError, Departed]] = ???
  }
}
```

To implement `Departures` interface we set boundaries for our effect `F`: it should have instances of `UUIDGen` and `Logger`.
In this application I've created dummy `UUIDGen` and `Logger` interfaces, don't use them in your multi-billion startup - you can find something better.    
Also `F` should have a `Monad` instance to chain function calls.  
Let's start by implementing a validation logic that will check if `Departure` is valid.
We will have only 1 check - if the destination city is in the list of connected cities.
```scala
def validated(departure: Departure)(f: F[Departed]): F[Either[DepartureError, Departed]] = {
  val destination = departure.to.city

  connectedTo.find(_ === destination) match {
    case None =>
      val e: DepartureError = DepartureError.UnexpectedDestination(destination)
      F.error(s"Tried to departure to an unexpected destination: $departure")
       .as(e.asLeft)
    case _ =>
      f.map(_.asRight)
  }
}
```
If the destination city is not on the list, we log an error message and return the error as the result. 
Otherwise, we create a `Departed` event and return it as the result.  
Let's see how `register` function can be implemented:
```scala
def register(departure: Departure): F[Either[DepartureError, Departed]] =
  validated(departure) {
    F.newEventId
      .map {
        Departed(
          _,
          departure.id,
          From(city),
          departure.to,
          departure.time,
          departure.actual.toTimestamp
        )
      }
      .flatTap(producer.send_)
  }
```
We start by validating the destination city. 
If it is valid then we generate a `newEventId` which we use to create a new `Departed` event.
This event is published to the `city` topic in Pulsar using `producer` that we passed to `make` function.
Simple!
You can find the final version of `Departures` [here](https://github.com/psisoyev/train-station/blob/ec3841784841ebc03c6d1cdc3347b04065e81d1c/service/src/main/scala/com/psisoyev/train/station/departure/Departures.scala#L13).  

## Expecting departed trains
We have learned how to spawn trains in our system. 
If a train has departed from Zurich to Bern, then Bern should be notified about it.  
Bern is listening for Zurich events and as soon as there is a `Departed` event with Bern set as the destination, it should add it to the expected train list. 
Let's leave message consuming for dessert and now will focus on business logic.
We define a `DepartureTracker` that expects a `Departed` event: 
```scala
trait DepartureTracker[F[_]] {
  def save(e: Departed): F[Unit]
}
```
This service will be a sink in our the `Departed` event flow, so we don't care about the return type and we don't expect any validation errors.
As we did with `Departures`, we create a companion object, where we define a `make` function: 
```scala
def make[F[_]: Applicative: Logger](
    city: City,
    expectedTrains: ExpectedTrains[F]
  ): DepartureTracker[F] = new DepartureTracker[F] {
    def save(e: Departed): F[Unit] =
      if (e.to.city =!= city) F.unit
      else
        expectedTrains.update(e.trainId, ExpectedTrain(e.from, e.expected)) *>
          F.info(s"$city is expecting ${e.trainId} from ${e.from} at ${e.expected}")
  }
```
We have a dependency on `ExpectedTrains` service. 
This service is the storage of incoming trains. 
We will implement it shortly. 
Here we have `save` function, which checks that the destination city of the incoming train is the expected one. 
For example, both Geneva and Zurich are consuming events from Bern. When Bern emits a `Departed` event, one city will just ignore the message.       
The other city, which is the destination city, will update the expected train list and create a log message.

Our expected train storage has a minimal set of functionality:
```scala
trait ExpectedTrains[F[_]] {
  def get(id: TrainId): F[Option[ExpectedTrain]]
  def remove(id: TrainId): F[Unit]
  def update(id: TrainId, expectedTrain: ExpectedTrain): F[Unit]
}
```
Even if we will try to remove a train that doesn't exist in our system, we will treat it as a non-failure. 
In some business cases it might be wrong and a sign of system malfunctioning, but in this particular case we will ignore that kind of errors.
For our MVP we will store data in memory without persisting it anywhere: 
```scala
def make[F[_]: Functor](
    ref: Ref[F, Map[TrainId, ExpectedTrain]]
  ): ExpectedTrains[F] = new ExpectedTrains[F] {
    def get(id: TrainId): F[Option[ExpectedTrain]] = 
      ref.get.map(_.get(id))
    def remove(id: TrainId): F[Unit] = 
      ref.update(_.removed(id))
    def update(id: TrainId, train: ExpectedTrain): F[Unit] = 
      ref.update(_.updated(id, train))
  }
```
We use [`Ref`](https://zio.dev/docs/datatypes/datatypes_ref) as our high-level concurrency mechanism. 

## Train arrival
The final part of the business logic trilogy is train arrival. 
Similarly to train departure, we create an HTTP endpoint, which we can call using a simple cURL POST request:
```
curl --request POST \
  --url http://localhost:8081/arrival \
  --header 'Content-Type: application/json' \
  --data '{
	"trainId": "123",
	"time": "2020-12-03T10:15:30.00Z"
}'
```
This request will be handled by Http4s routes: 
```scala
case req @ POST -> Root / "arrival" =>
  req
    .asJsonDecode[Arrival]
    .flatMap(arrivals.register)
    .flatMap(_.fold(handleArrivalErrors, _ => Ok()))
```
We call `register` method on a twin service of `Departures` service we've seen earlier - `Arrivals`.
`Arrivals` services also has only one method:
```scala
trait Arrivals[F[_]] {
  def register(arrival: Arrival): F[Either[ArrivalError, Arrived]]
}
```
Again we start with validation of the request:
```scala
def validated(arrival: Arrival)(f: ExpectedTrain => F[Arrived]): F[Either[ArrivalError, Arrived]] =
  expectedTrains
    .get(arrival.trainId)
    .flatMap {
      case None =>
        val e: ArrivalError = ArrivalError.UnexpectedTrain(arrival.trainId)
        F.error(s"Tried to create arrival of an unexpected train: $arrival")
         .as(e.asLeft)
      case Some(train) =>
        f(train).map(_.asRight)
    }
```
Here we check if the arrived train was expected and if it is, then we create an `Arrived` event. Otherwise, we create an error and log it.
If you take a look at the implementation you will notice similarities to the other `register` method we've seen earlier:
```scala
def register(arrival: Arrival): F[Either[ArrivalError, Arrived]] =
  validated(arrival) { train =>
    F.newEventId
      .map {
        Arrived(
          _,
          arrival.trainId,
          train.from,
          To(city),
          train.time,
          arrival.time.toTimestamp
        )
      }
      .flatTap(a => expectedTrains.remove(a.trainId))
      .flatTap(producer.send_)
  }
```
Comparing to `Departures` the difference is that we not only publish the new event but also have another side-effect - we remove the arrived train from the list of expected trains. 

That is all the business logic needed for the MVP. This logic is covered by unit tests and they are [available on GitHub](https://github.com/psisoyev/train-station/tree/ec3841784841ebc03c6d1cdc3347b04065e81d1c/service/src/test/scala/com/psisoyev/train/station). 
Unit tests are implemented using ZIO Test. If you want to learn more about it, you can check one of my previous articles - [Effective testing with ZIO Test](/zio-test/).  

# The dessert
Remember I said we'll leave message consuming for dessert? The time has come!  
In this section, we will wire all logical services together.

## Building resources
Let's start by creating the required resources. 
A city node needs 4 things: configuration, event producer, event consumers, and a `Ref` where we will store `ExpectedTrains`.
We can combine them in one case class and create outside of `Main` class:
```scala
final case class Resources[F[_], E](
  config: Config,
  producer: Producer[F, E],
  consumers: List[Consumer[F, E]],
  trainRef: Ref[F, Map[TrainId, ExpectedTrain]]
)
```
`Config` is read from environment variables. 
For this purpose, we will use a library called [ciris](https://github.com/vlovgr/ciris).
We won't focus on it too much, as the configuration is boring. 
You can find the final implementation of `Config` on [GitHub](https://github.com/psisoyev/train-station/blob/ec3841784841ebc03c6d1cdc3347b04065e81d1c/server/src/main/scala/com/psisoyev/train/station/Config.scala#L13).

Producers and consumers are much more interesting to see. 
To create them we will use library called [neutron](https://github.com/cr-org/neutron/) developed by [Chatroulette](https://about.chatroulette.com/).

First, we need to establish a connection with Apache Pulsar cluster. 
For this, we create an instance of `Pulsar` object:
```scala
Pulsar.create[F](config.pulsar.serviceUrl)
```
As we can see, it requires only a `serviceUrl`. In return, we will get a `Resource[F, PulsarClient]`.
This resource can be used to create producers and consumers.
Before we create a producer, we need to create a `Topic` object, which contains topic configuration:
```scala
def topic(config: PulsarConfig, city: City) =
  Topic(
    Topic.Name(city.value.toLowerCase),
    config
  ).withType(Topic.Type.Persistent)
```
The topic name simply will be a city name and it will be a `Persistent` topic. So that we don't lose any unacknowledged messages.
Also, as a part of the configuration, we pass `namespace` and `tenant`. 
You can learn more about these concepts [in the official documentation](https://pulsar.apache.org/docs/en/concepts-multi-tenancy/).

Creating a producer is a simple one-liner:
```scala
def producer(client: Pulsar.T, config: Config): Resource[F, Producer[F, E]] =
  Producer.create[F, E](client, topic(config.pulsar, config.city))
```
There are several ways of creating a producer, we'll use the simplest one. 
It requires only Pulsar client we created earlier and a topic.

Creating a consumer requires slightly more actions, as we also have to create a `Subscription`:
```scala
def consumer(client: PulsarClient, config: Config, city: City): Resource[F, Consumer[F, E]] = {
  val name         = s"${city.value}-${config.city.value}"
  val subscription = Subscription(Subscription.Name(name)).withType(Subscription.Type.Failover)

  Consumer.create[F, E](client, topic(config.pulsar, city), subscription)
}
```
We create a subscription with a name corresponding to a connected city name plus the city we'd like to run. 
By default, we will use the `Failover` subscription type as it allows us to run 2 instances in parallel, just in case one will go down.

Together with the required `Ref` we can finally build our `Resources`:
```scala
for {
  config    <- Resource.liftF(Config.load[F])
  client    <- Pulsar.create[F](config.pulsar.serviceUrl)
  producer  <- producer(client, config)
  consumers <- config.connectedTo.traverse(consumer(client, config, _))
  trainRef  <- Resource.liftF(Ref.of[F, Map[TrainId, ExpectedTrain]](Map.empty))
} yield Resources(config, producer, consumers, trainRef)
```
Note, that we create a list of consumers using `traverse` method on `connectedTo` list of cities. 
As always, you can find the final result [on GitHub](https://github.com/psisoyev/train-station/blob/b7447c40f88e19020c33f799bcbb9c5b94a7d5ac/server/src/main/scala/com/psisoyev/train/station/Resources.scala#L11).

## Starting the engine
We will use `zio.Task` as the effect type. 
It contains the least amount of type parameters so should be easier to understand for those, who are not familiar with ZIO.  
However, if you want to see a few more type parameters in action, you can read my [introduction to ZIO](/welcome-zio/).

First, we create the `Resources` class we've defined earlier:
```scala
Resources
  .make[Task, Event]
  .use {
    case Resources(config, producer, consumers, trainRef) => ???
  }
``` 
Nothing has changed here - same 4 parameters as before. 
We start with initializing services and creating `routes` for the HTTP server:
```scala
val expectedTrains   = ExpectedTrains.make[Task](trainRef)
val arrivals         = Arrivals.make[Task](config.city, producer, expectedTrains)
val departures       = Departures.make[Task](config.city, config.connectedTo, producer)
val departureTracker = DepartureTracker.make[Task](config.city, expectedTrains)

val routes = new StationRoutes[F](arrivals, departures).routes.orNotFound
```
Then we create the HTTP server:
```scala
val httpServer = Task.concurrentEffectWith { implicit CE =>
  BlazeServerBuilder[Task](ec)
    .bindHttp(config.httpPort.value, "0.0.0.0")
    .withHttpApp(routes)
    .serve
    .compile
    .drain
}
```
Nothing new here for those, who are already familiar with Http4s. Who is not familiar with Http4s, I encourage you to [read the docs](https://http4s.org/).
The next step is to start consuming incoming messages and to build a stream out of them:
```scala
val departureListener =
  Stream
    .emits(consumers)
    .map(_.autoSubscribe)
    .parJoinUnbounded
    .collect { case e: Departed => e }
    .evalMap(departureTracker.save)
    .compile
    .drain
```
Let's take a closer look at what is going on here. 
We are using FS2 library to create a stream of events.
First, we create a stream of consumers and call `autoSubscribe` method on every consumer.
This will start a subscription on the topic. Then we merge all streams into one stream using `parJoinUnbounded`.
After that, we remove every other message except `Departed` using `collect` method.
The final step is to call `save` method on our `departureTracker` which we implemented earlier.
Then the stream gets compiled and drained.

Now we have 2 final streams: HTTP server and incoming Pulsar messages.
At this point we are already handling all the messages and simply need to run the streams, so we just zip them in parallel and drop the result: 
```scala
departureListener
  .zipPar(httpServer)
  .unit
```
That is it. Our `Main` class consists of a few simple blocks of code which we can easily read and maintain.

# Summary
Thank you for reading up to this point. 
In this article, I've given you an example of an event-driven system. 
Together we've built a Swiss railway network simulator MVP. 
Of course, some decisions and choices can be challenged. 
You must remember that this is not a tutorial on how you *should* build your next multi-billion startup, but how you *can* build one.

We've seen some capabilities of Apache Pulsar, but I promise there is much more than that. 
The system has really surprised me with its simplicity and capabilities. 
We've built a simple but distributed system consisting of several nodes that are communicating using messaging on top of Apache Pulsar.
All of this on the functional stack using Tagless Final approach on top of `cats` library, where `ZIO` `Task` is used as a main effect type.  
Also, we've tried [Neutron](https://github.com/cr-org/neutron/) which is still in active development but is used in production by [Chatroulette](https://about.chatroulette.com/).

The final version of the app is available on [GitHub](https://github.com/psisoyev/train-station/). 
You can find instructions on how to run it in the readme of the project. 

Let me know if you have any questions or suggestions in the comment section! Feedback is also very welcome, thank you!
Cheers!