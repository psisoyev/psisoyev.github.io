---
layout: post
title: Effective testing with ZIO Test [RC17 edition]
date: 2020-01-20 13:37:00 +0100
description: Effectful testing with ZIO Test
img: test/test.png
tags: [Scala, ZIO, ZTest, unit testing, rc17]
comments: true
---
This article will help you to effectively test your "effectful" ZIO code.

In the [previous article](https://scala.monster/welcome-zio/) we were exploring ZIO. 
We've built the [release pager](https://scala.monster/design-a-pager/) application but we have skipped something very important - unit tests.
In this article, we will continue to develop the application and we will write tests for it.
Full versions of the code snippets I will use in this article are available on [GitHub](https://github.com/psisoyev/release-pager/tree/chapter1/service/src/test/scala/io/pager).    

**Warning! This blog post is outdated and is based on an old ZIO version `1.0.0-RC17`.** 
Please [check out the updated post with the new ZIO test features](https://scala.monster/zio-test/). 

ZIO has its own ecosystem and provides developer tools to increase development efficiency.
One of the things which are included in the ZIO toolbox is ZIO Test framework.
ZIO Test is designed for "effectful" testing.

The testing topic is broad and to stay specific we will not cover these questions:
* "What is ZIO?"
* "What are effects?"
* "What is unit testing?" 
* "Why do we need unit tests?"
* "ZIO Test vs AngularJS"

Instead, let's focus on:
* "How can we test effects with ZIO Test?"
* "How can we mock services with ZIO Test?"
* "How can we write property based tests with ZIO Test?"
* "What common issues we can have using ZIO Test for the first time?"
* "What are additional capabilities ZIO Test can provide?"

## Getting started with ZIO Test 
We start with adding test dependencies in [Dependencies.scala](https://github.com/psisoyev/release-pager/blob/chapter1/project/Dependencies.scala):
```scala
val zioTest    = "dev.zio" %% "zio-test"     % Version.zio % "test"
val zioTestSbt = "dev.zio" %% "zio-test-sbt" % Version.zio % "test"
``` 
As we are planning to run the tests with SBT we have to specify a special test framework in SBT [Settings.scala](https://github.com/psisoyev/release-pager/blob/chapter1/project/Settings.scala#L28):
```scala
testFrameworks := Seq(new TestFramework("zio.test.sbt.ZTestFramework"))
```
Now we are ready to start writing test scenarios. 
We will start with functionality, that doesn't have dependencies on other services. 
As you remember we have implemented the [SubscriptionLogic service.](https://github.com/psisoyev/release-pager/blob/chapter1/service/src/main/scala/io/pager/subscription/SubscriptionLogic.scala)
Let's test it!

We create a `LiveSubscriptionLogicSpec.scala` [file](https://github.com/psisoyev/release-pager/blob/chapter1/service/src/test/scala/io/pager/subscription/LiveSubscriptionLogicSpec.scala) in `io.pager.subscription` package in `test` folder. 
```scala
import zio.test._
import io.pager.subscription.SubscriptionLogicTestCases._

object LiveSubscriptionLogicSpec extends DefaultRunnableSpec(suite(specName)(scenarios: _*))

object LiveSubscriptionLogicTestCases {
  type TestScenarios = List[ZSpec[Random with Sized, Throwable, String, Unit]]

  val specName: String = "LiveSubscriptionLogicSpec"
  val scenarios: TestScenarios = List()
}
```
To start, we import `zio.test` package. This package contains ZIO Test building blocks.
We extend `DefaultRunnableSpec` trait which is similar to `zio.App` - it provides `ZEnv` (Random, Clock etc.) and runs provided scenarios.
The only parameter we will pass to `DefaultRunnableSpec` is a spec. 
To create a spec we use `suite` method, which has two parameters - suite name and test scenarios.
We have extracted both parameters in a singleton object, which we have to import on top of the file.
We could avoid splitting `LiveSubscriptionLogicSpec` into two objects but it looks cleaner when test scenarios are defined outside of the `DefaultRunnableSpec` class arguments.
From now we can run our unit tests by clicking `Run` button near the class name in IntelliJ IDEA or using `sbt test` command.

##### Note: After ZIO 1.0 release this split won't be needed anymore as the syntax will look a bit differently. 

We have defined our first spec. Currently it is empty, let's fix that.
[Last time](https://scala.monster/welcome-zio/) we have implemented `SubscriptionLogic`. 
Main task of it is to handle user subscriptions to GitHub repositories.  
As we are writing unit tests we would like to abstract from the database and will store everything in memory. 
Fortunately, we already have implemented in-memory versions of `ChatStorage` and `RepositoryVersionStorage`.    

We start with creating an instance of the service, which we will be used in every scenario:
```scala
type RepositoryMap   = UIO[Ref[Map[Name, Option[Version]]]]
type SubscriptionMap = UIO[Ref[Map[ChatId, Set[Repository.Name]]]]

private def service(
  subscriptionMap: SubscriptionMap = emptyMap[ChatId, Set[Name]],
  repositoryMap: RepositoryMap = emptyMap[Name, Option[Version]]
): UIO[SubscriptionLogic.Service[Any]] =
  for {
    repositoryMap   <- repositoryMap
    subscriptionMap <- subscriptionMap
  } yield {
    val chatStorage              = ChatStorage.Test.make(subscriptionMap)
    val repositoryVersionStorage = RepositoryVersionStorage.Test.make(repositoryMap)
  
    SubscriptionLogic.Live.make(
      logger = Logger.Test,
      chatStorageService = chatStorage,
      repositoryVersionStorageService = repositoryVersionStorage
    )
  }
```
In order to instantiate the service, we have to provide an initial in-memory state. 
We express the state using `Map` collection wrapped in `zio.Ref`.
By default, these maps are empty and do not contain any data.
However, in more advanced test cases we can prepare the state before we write some actual test scenarios.
The state is used to instantiate storage services, which are used in `SubscriptionLogic.Live` service. 
To keep things simple we replace logger with a dummy instance. 
Ideally, we would like to test log messages as well.

We will be clever and will use property-based tests. For that we have to write [data generators](https://github.com/psisoyev/release-pager/blob/chapter1/service/src/test/scala/io/pager/Generators.scala):
```scala
import zio.random.Random
import zio.test.{Gen, Sized}
import zio.test.Gen._

object Generators {
  val repositoryName: Gen[Random with Sized, Name] = anyString.map(Name.apply)
  val chatId: Gen[Random with Sized, ChatId] = anyLong.map(ChatId.apply)
}
```
We are reusing generators provided by the framework. Generated values are wrapped into our values classes.
Both generators require `Random` and `Sized` instances and ZIO will provide them in `ZEnv`. 

It is time to implement our first test scenario:
```scala
val scenarios = List(
  testM("successfully subscribe to a repository") {
    checkM(repositoryName, chatId) { case (name, chatId) =>
      for {
        service       <- service()
        _             <- service.subscribe(chatId, name)
        repositories  <- service.listRepositories
        subscriptions <- service.listSubscriptions(chatId)
        subscribers   <- service.listSubscribers(name)
      } yield {
        assert(repositories, equalTo(Map(name -> None))) // there might be something missing here
        assert(subscriptions, equalTo(Set(name)))        // and here ...
        assert(subscribers, equalTo(Set(chatId)))
      }
    }
  }
)
```
Let's take a closer look at what is going on here.
`testM` builds a test scenario. We label this test and in the body of it, we have an effectful test. 
There is also a pure version available - it's simply `test`.

`checkM` provides the test with generated data samples. In this test we have random repository name and random chat id.
If we would need to generate more than one `chatId` we would have to specify 2 generators:
```scala
checkM(repositoryName, chatId, chatId) { (name, chatId1, chatId2) =>
```
However, ZIO Test has a limit of 4 generators in one `checkM` call. If we would need more than that, we could have several `checkM` calls inside the test or combine 2 generators into one:
```scala
val chatIds: Gen[Random with Sized, (ChatId, ChatId)] = chatId <*> chatId
```
The "[TIE fighter](https://en.wikipedia.org/wiki/TIE_fighter)" operator is `zip` method which will create a tuple of the generators. We can use it in the test:
```scala
checkM(repositoryName, chatIds) { (name, (chatId1, chatId2)) =>
```
The scenario itself is a for comprehension which describes calls to the subscription service and compares expected results with actual.
To compare the results we use `assert` method which has two parameter lists: the actual value and the expected value.
If we would like to do the same for effectful values, we would call `assertM` method.
ZIO provides different expectations that you can use to build your assertions: `equalTo`, `isLessThen`, `contains`, `isTrue` and many others.
You can even compose these assertions together, eg. `isRight(isSome(equalTo(1)))`
Names of the expectations should give us a hint about what they are doing. If you would like to learn more please check the [Scaladoc](https://github.com/zio/zio/blob/master/test/shared/src/main/scala/zio/test/Assertion.scala).
Provided expectations can be used to build your custom expectations.

## The biggest trap I have fallen with ZIO Test.
This kind of mistakes might happen not only when working with ZIO Test, but with any other effectful code.
I was used to `ScalaTest` matchers and wrote all the assertions in a column. 
However, assertions are values. 
ZIO Test assertions do not have side effects.
Do you see where I'm getting? 
Assertions won't throw an exception and abort a test if assertion has failed.
Assertions must be chained and checked by the framework. 
It means that in the test above only 1 out of 3 assertions is checked. 
Unfortunately, these mistakes happen and the compiler won't guard you.
Hopefully, it will change in the future.  
That's definitely not the behavior we would expect. The correct version would be: 
```scala
  ...
} yield {
  assert(repositories, equalTo(Map(name -> None))) &&
  assert(subscriptions, equalTo(Set(name))) &&
  assert(subscribers, equalTo(Set(chatId)))
}
```
The only difference with the original code are the added `&&` (AND) operators. 
ZIO Test assertions are combined together using boolean algebra operators.
In other cases, we could use `||` (OR) operator. Also, we can negate the assertion using `!` (exclamation mark).
For those who don't like symbolic notations, there are named versions of the operators. 
I encourage you to check the [code](https://github.com/zio/zio/blob/master/test/shared/src/main/scala/zio/test/BoolAlgebra.scala) to get a deeper knowledge of the available operations.

To avoid that kind of mistakes I would recommend you trying a linter. 
Code reviews might also help but these mistakes are quite hard to spot. 
In this specific project I have used [Wartremover](https://www.wartremover.org/).
It's a Scala linter, that has a flag `NonUnitStatements` which will help you to find unused effects at compilation step. 
Unfortunately, linters have their own drawbacks. 
`Wartremover` caught quite a lot of false positives and we have to exclude some of the checks.
You could also try using other linters, that look for non-unit statements. 
Also, if you like experimental stuff you could try [ZIO shield](https://github.com/zio/zio-shield).

## Fake all the work!
Let's start with a simple example. 
This example is taken from [ZIO codebase](https://github.com/zio/zio/blob/24d94f14256a28b2ccf130f0eea7c3c5620528a8/examples/shared/src/test/scala/zio/examples/test/MockingExampleSpec.scala#L36).
```scala
testM("expect call for overloaded method") {
  val app     = random.nextInt
  val mockEnv = MockRandom.nextInt._1 returns value(42)
  val result  = app.provideManaged(mockEnv)
  assertM(result)(equalTo(42))
}
```
Value `app` is of type `ZIO[Random, Nothing, Int]`. 
Here `Random` is a required environmental dependency.  
In other cases, it might be a resource, for example, a SQL database connection. 
Even though we don't use environmental dependencies in the `release-pager` yet, we could have one later. 
We simulate `Random` using a mocked instance and provide it to the `app` as a managed resource.
Also, if we would have a resource that we would like to share between scenarios, we would provide it once with `provideManagedShared` method of `Spec`:
```scala
object WithResourceSpec extends DefaultRunnableSpec(testSuite)

object WithResourceSpecTestCases {
  val specName: String = "WithResourceSpec"
  val scenarios = Seq(...)
  val myExpensiveResource: UManaged[ExpensiveResource] = ...

  val testSuite = suite(specName)(scenarios: _*)
    .provideManagedShared(myExpensiveResource)
}
```
Let's have a look at more specific and advanced test examples.
We have a `ReleaseChecker` [service](https://github.com/psisoyev/release-pager/blob/chapter1/service/src/main/scala/io/pager/lookup/ReleaseChecker.scala), which has dependencies on other services. 
To be super-efficient we don't want to build the whole dependency tree. 
We want to test the reaction of `ReleaseChecker` for different inputs.
In this specific case, the service is gathering some of the inputs from other services.
We would like to fake these services and provide different kinds of inputs ourselves.

In the [previous post](https://scala.monster/welcome-zio/) we were not using the environment part of the ZIO data type. 
It is time to change that. 
We will adopt the module pattern.
Please check the [official documentation](https://zio.dev/docs/howto/howto_use_module_pattern) to get a deep understanding of the pattern and background behind it.
In short, we will structure our services as modules. `Service` traits will have an environment type parameter, which we will use in testing.
The downside of the approach is that in most cases we will clutter signatures with `Any` as environment parameter:
```scala
def gitHubClient: GitHubClient.Service[Any]
def telegramClient: TelegramClient.Service[Any]
def subscriptionLogic: SubscriptionLogic.Service[Any]
```
From the docs:
##### By convention, we name the value holding the reference to the service the same as a module, only with first letter lowercased. This is to avoid name collisions when mixing multiple modules to create the environment.

In the case of `SubscriptionLogic` we will have the module definition shown below:
```scala
trait SubscriptionLogic {
  val subscriptionLogic: SubscriptionLogic.Service[Any]
}

object SubscriptionLogic {
  trait Service[R] {
    def subscribe(chatId: ChatId, name: Name): RIO[R, Unit]
      ... // skipping the rest
  }
  
  trait Live extends SubscriptionLogic {
    def logger: Logger.Service
    def chatStorage: ChatStorage.Service
    def repositoryVersionStorage: RepositoryVersionStorage.Service

    override val subscriptionLogic: Service[Any] = new Service[Any] {
      override def subscribe(chatId: ChatId, name: Name): Task[Unit] =
        ... // skipping the rest
    }
}
```
Service has environment parameter `R` that in case of `Live` implementation is substituted by `Any` as we don't actually have environmental dependencies there.
All service dependencies are defined as functions, that will be overridden when we will wire services together.    

ZIO mocks will be defined inside of the `SubscriptionLogic` companion object.
We need 2 things: service method definitions and a mockable instance.
Method definitions are described in `Service` companion object. 
For every service method, we create an object which extends `zio.test.mock.Method` trait with three parameters: module, method inputs, and method return type.
We will see how to mock the `subscribe` method.
```scala
object Service {
  object subscribe extends Method[SubscriptionLogic, (ChatId, Name), Unit]
    ... // skipping the rest
}
```
The mockable instance is mocked implementation of the service. 
It will be used in our test scenarios. 
This instance must be placed inside of the `SubscriptionLogic` so that it can be automatically discovered.  
```scala
implicit val mockable: Mockable[SubscriptionLogic] = (mock: Mock) =>
    new SubscriptionLogic {
      val subscriptionLogic = new SubscriptionLogic.Service[Any] {
        def subscribe(chatId: ChatId, name: Name): UIO[Unit] = mock(Service.subscribe, (chatId, name))
          ... // skipping the rest
```
Good news! We have mocked the first method in the service.
Unfortunately, there are some bad news as well: we have to implement interaction with the mock instance for every method. 
Sounds boring.
However, some people say "work smarter, not harder".
Guys from the ZIO community were inspired by these words and have created [zio-macros project](https://github.com/zio/zio-macros). 
It can help you to remove unnecessary boilerplate from your ZIO projects. 
Including these nasty macro definitions.
```scala
val zioMacroTest = "dev.zio" %% "zio-macros-test" % Version.zioMacro
```
We add the dependency to the project and we are ready to go. 
We will need only one annotation - `@mockable`. 
I would advise you to check out the project link above as there are other macros that might simplify your life.

To get rid of the boilerplate using `@mockable` we have to follow the module pattern and mark the module definition with the annotation:
```scala
@mockable
trait SubscriptionLogic {
  ...
}
```
Aaaand... that's it!
From now on we can use `SubscriptionLogic` mocks in our tests. Let's test the `ReleaseChecker` service, which depends on `SubscriptionLogic`. 
Also, it has few other dependencies that we have to mark with the `@mockable` annotation. 
We create a [new spec](https://github.com/psisoyev/release-pager/blob/chapter1/service/src/test/scala/io/pager/lookup/LiveReleaseCheckerSpec.scala):
```scala
object LiveReleaseCheckerSpec extends DefaultRunnableSpec(suite(specName)(scenarios: _*))

object LiveReleaseCheckerTestCases {
  val specName: String = "LiveReleaseCheckerSpec"

  val scenarios: TestScenarios = List()
}
```
As we remember, `ReleaseChecker` has one method - `scheduleRefresh`. 
This method contains quite a lot of logic:
1. it looks for GitHub repositories with subscribers
2. takes latest repository versions
3. checks for new repository versions
4. if there are no new versions it finishes or if there are new versions it updates the version in the storage and 
5. looks for repository subscribers
6. notifies subscribers about the new version

Whoah. We could split that into few methods and test them separately. 
However, let's imagine we have a good reason to have such a big method.
In every scenario, we will have to call the same method of `ReleaseChecker`.
The difference between test scenarios will be only in mock behavior. 
We can create a method, which builds the service using mocks and calls the `scheduleRefresh` method.
We will re-use this method in several tests of the test suite.
```scala
def scheduleRefreshSpec(
    subscriptionMocks: UManaged[SubscriptionLogic],
    telegramClientMocks: UManaged[TelegramClient],
    gitHubClientMocks: UManaged[GitHubClient]
  ): Task[TestResult] = {
    (subscriptionMocks &&& telegramClientMocks &&& gitHubClientMocks)
      .map { case ((sl, tc), gc) => ReleaseChecker.Live.make(Logger.Test, gc, tc, sl) }
      .use(_.scheduleRefresh)
      .as(assertCompletes)
  }
```
Every mock is an `Expectation` that can be converted to `Managed` resource. 
`UManaged` is an alias to `Managed`, which doesn't have an environment and doesn't have an error.
We combine mocks using `&&&` (zip), map over them and build the service. 
The service is also a resource now. We can `use` it and call `scheduleRefresh` method. 
As the method doesn't return us anything we say that we expect the test to be successful every time (`assertCompletes`).
By the way - `as` is just a `map` alias, that drops the result of the previous effect.

Finally, let's implement a test scenario with mocked services. 
Below you can see one of the test scenarios.
```scala
testM("Notify users about a new release") {
  checkM(repositoryName, chatIds) { (name, (chatId1, chatId2)) =>
    val repositories = Map(name -> Some(Version("0.0.1-RC17")))
    val subscribers  = Set(chatId1, chatId2)
    val msg = message(name)

    val gitHubClientMocks = GitHubClient.releases(equalTo(name)) returns value(releases)
    val telegramClientMocks = TelegramClient.broadcastMessage(equalTo((subscribers, msg))) returns unit
    val subscriptionLogicMocks =
      (SubscriptionLogic.listRepositories returns value(repositories)) *>
      (SubscriptionLogic.updateVersions(equalTo(Map(name -> finalVersion))) returns unit) *>
      (SubscriptionLogic.listSubscribers(equalTo(name)) returns value(subscribers))

    scheduleRefreshSpec(subscriptionLogicMocks, telegramClientMocks, gitHubClientMocks)
  }
}
```
It's a big example, let's look at it closely.
```scala
val gitHubClientMocks = GitHubClient.releases(equalTo(name)) returns value(releases)
```
As we don't want to call a real GitHub service we tell the mock to return us the list of releases.
`GitHubClient.releases` is one of the methods magically generated by `zio-macros` (or manually written by us if we decided to stay macro-free).
We say that whenever it accepts a value which equals to `name` we return a [pre-defined list](https://github.com/psisoyev/release-pager/blob/9ada7390d82397b47df6eec86861ea445412ff63/service/src/test/scala/io/pager/TestData.scala) of releases.   
Looks simple. We just have to get used to the assertions (e.g `equalTo`), which are pretty straight-forward.

`telegramClientMocks` should be simple to understand if you got the idea of `gitHubClientMocks`. 
There are only `subscriptionLogicMocks` left. 
```scala
val subscriptionLogicMocks =
  (SubscriptionLogic.listRepositories returns value(repositories)) *>
  (SubscriptionLogic.updateVersions(equalTo(Map(name -> finalVersion))) returns unit) *>
  (SubscriptionLogic.listSubscribers(equalTo(name)) returns value(subscribers))
```
Here we have several calls of the service.
As it was mentioned, a method call on a ZIO mock is an `Expectation`. 
This means, that if the expected method wasn't called, the framework will return us an error that will fail the test. 
Also, a test will fail if we have called a method with unexpected arguments. 
Basically, the framework gives us the possibility to set strict expectations and tests will fail when something unexpected happens. 
As we have several expectations of the mock we must chain these expectations using `flatMap` or the "ice-cream operator" `*>`.
Note, that expectations are not associative. This means that we must keep the correct mocked method call order.

Finally, we pass the mocks to previously defined `scheduleRefreshSpec` method and we are done. 
We have a small and concise scenario that tests a relatively big method.

There are several `ReleaseChecker` test cases available on [GitHub](https://github.com/psisoyev/release-pager/blob/9ada7390d82397b47df6eec86861ea445412ff63/service/src/test/scala/io/pager/lookup/LiveReleaseCheckerSpec.scala).
You are welcome to review them.

### One little issue with mock support in some IDEs
If you are using [Metals](https://scalameta.org/metals/) you are safe and you can skip this part.
However, if you are using IntelliJ IDEA there is a small issue.
Even if you have installed [zio-intellij plugin](https://github.com/zio/zio-intellij) you won't see all macro generated code and IntelliJ will treat this code as erroneous:
![IDEA errors]({{site.baseurl}}/assets/img/test/idea-errors.png#center)

In the meantime, the same code in VSCode with Metals has no errors and auto-complete works fine:
![Metals]({{site.baseurl}}/assets/img/test/metals-no-errors.png#center)

Unfortunately, `@mockable` annotation is not yet supported by `zio-intellij`. 
Even though other annotation (e.g `@accessible`) generated code is discoverable. 
I have created [an issue](https://github.com/zio/zio-intellij/issues/28) to support `@mockable`. 
Of course, if you will write mocks manually this issue won't bother you.

### Bonus track
ZIO Test provides users with test aspects. 
You can think of these aspects like test features or traits.
There are some generic features that you would like to use in your tests.
Here are some of them: 
* `ignore` - mark test as ignored
* `after` - runs an effect after a test
* `flaky` - retries a test until success
* `nonFlaky` - repeats a test `n` times to ensure it is stable
* `jvmOnly` - runs a test only on JVM platform 
* and [others](https://github.com/zio/zio/blob/417ffbcfcc9aceee1abe731be9ab98df0ffd0638/test/shared/src/main/scala/zio/test/TestAspect.scala#L97).

Aspects can be used both on scenario level and on suite level. 
```scala
import TestAspect._

object LiveReleaseCheckerSpec extends DefaultRunnableSpec(suite(specName)(scenarios: _*), List(nonFlaky))

object LiveReleaseCheckerTestCases {
  val specName: String = "LiveReleaseCheckerSpec"

  val scenarios: TestScenarios = List(
    testM("Do not call services if there are no repositories") {
      ...
    } @@ ignore,
```
We marked all suite tests as `nonFlaky` and the first test in the suite is marked as ignored. 
But be careful, blindly marking all tests as `nonFlaky` will affect your test performance. 
By default, framework will run the spec for 100 times.

## Summary 
In this article we have explored ZIO Test and its capabilities:
* we used building blocks from `zio.test` to construct a spec
* we replaced storage state with an in-memory implementation, which we have constructed in a test
* we applied property-based testing and created data generators
* we learned how to compare expected results with actual results using `assert`
* we avoided common mistake with effect chaining
* we touched resource mocking
* we built service call expectations using mocks
* we mocked all the service dependencies
* we removed all the boilerplate required for service mocking using macros
* we saw what issues we can expect from using macros and how to handle them
* we added test features using aspects

That's a lot! The best part of this is that there are more things that you can explore by yourself: 
resource management (we saw just a little example), ScalaJS support, etc. 

## Conclusion
I like how the code in ZIO Test is well organized. You can easily find tools that you need to build a test.
There are no magical implicit conversions, that you have to remember to import. 
There are no macros in the framework. 
However, you have to write some boilerplate, that will clutter your code. 
Of course, there is an alternative - to use macros from a separate project, which is not part of the original framework.
Having no macros might benefit you in long-term, when you will move your code to Scala 3.
As we are working with effects, we always have async runtime and we never use blocking in tests.
Tests written with ZIO Test are isolated, but can use shared resources without heavy machinery involved.
The framework makes sure that resources are closed properly.
Finally, there are test aspects, that provide you with ways to add generic test features.

However, there are some things, that can be improved. 
I would like to underline the fact, that at the moment of writing, ZIO is not yet officially released. 
There are still some [open issues](https://github.com/zio/zio/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen) on GitHub. 
To be fair, there are quite many open issues, but there are only few, that are blockers for the official release.

If you try to use the framework yourself, you could have a feeling that there is missing tooling. 
It might be that in some exotic use cases you will miss some of the testing framework functionality.
But look at it from the other side -  you have a chance to get a hands on experience in development of open source framework. 
This is your opportunity to drive the direction of this specific technological ecosystem.

It really feels that ZIO community is investing a lot to make it easy for new users to onboard and start using the framework. 
There are many [other projects](https://github.com/zio) in the ecosystem that are actively developed. 
I'm happy to continue my journey through the ecosystem. 

If you are interested in getting notifications whenever new article is out - [follow me on Twitter](https://twitter.com/scalamonster).
Tune in!