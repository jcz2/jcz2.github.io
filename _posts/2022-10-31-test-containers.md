---
layout: post
title: 'Writing tests with testcontainers-scala'
date: 2022-10-31 16:57:22 +0100
tags: scala testing containers
---

Let's say we want to write API tests for a backend service.
The service uses MongoDB database to store the data.
We want tests to run against a setup that is as close to the real thing as possible.
Therefore we would like to avoid mocking the connection to the database and use an
actual MongoDB instance.
The most straightforward approach is to use a dockerized instance, start it before the tests
and clean it up afterward.
We can achieve that with [testcontainers-scala](https://github.com/testcontainers/testcontainers-scala).

testcontainers-scala library is a Scala wrapper around java test containers.
It makes running containers during tests straightforward and integrates with scalatest
by exposing `ForEachTestContainer` and `ForAllTestContainer` traits.

Let's see how we can use it to test our service.
The app that we will test is a simple todo service written with [http4s](https://http4s.org/) that exposes three endpoints to list, create and delete todos.

```scala
GET /todos
POST /todos
DELETE /todos/:id
```

We start by creating a scalatest spec and extend the usual traits:

```scala
import org.scalatest.funspec.AnyFunSpec
import org.scalatest.matchers.should.Matchers

class TodoServiceSpec extends AnyFunSpec with Matchers {
}
```

Then we extend one of the testcontainers specific traits.
There are two `ForEachTestContainer` which starts a new container before each test in a spec
and `ForAllTestContainer` which creates one container for the entire spec.
We'll go with the first one since we want to isolate each test.

```scala
import org.scalatest.funspec.AnyFunSpec
import org.scalatest.matchers.should.Matchers

class TodoServiceSpec extends AnyFunSpec with Matchers with ForEachTestContainer {
  override val container: GenericContainer = ???
}
```

Once we extend the trait we have to provide a value for the `container` field.
One issue I've run into is that `container` is originally defined as `def`, not `val`
but if you implement it as such the container won't start properly.

testcontainers-scala provides a lot of predefined containers for commonly used
services like MySQL, Kafka, or Mongodb which makes it even simpler to run these services.
I will use the `GenericContainer` here for demonstration purposes.

```scala
override val container: GenericContainer = GenericContainer(
  "mongo:5.0",
  exposedPorts = Seq(27017),
  waitStrategy = Wait.defaultWaitStrategy(),
)
```

There are a few more options but these are the most important ones.
The image name, exposed port, and wait strategy.
This is another place where I've run into issues.
The thing is that the test containers library will map the provided
port `27017` to a random port on the host.
We can access this random port with `mappedPort`

```scala
val host = container.containerIpAddress
val port = container.mappedPort(27017)
```

To avoid potential issues with CI, we are also getting the container ip address.

With this setup we can write our tests:

```scala
class TodoServiceSpec
  extends AnyFunSpec
  with ForEachTestContainer
  with Matchers
{

  override val container: GenericContainer = GenericContainer(
    "mongo:5.0",
    exposedPorts = Seq(27017),
    waitStrategy = Wait.defaultWaitStrategy()
  )

  def getTodoService: TodoService = {
    val host = container.containerIpAddress
    val port = container.mappedPort(27017)

    val mongoClient = new MongoClient(host, port)
    val todoDAO = new TodoDAO(mongoClient)
    new TodoService(todoDAO)
  }

  def getBody[T](response: Response[IO])
    (implicit entityDecoder: EntityDecoder[IO, T]): T = response
    .as[T]
    .unsafeRunSync()

  def makeRequest(
    todoService: TodoService,
    request: Request[IO]
  ): Response[IO] = todoService.routes
    .run(request)
    .unsafeRunSync()

  describe("TodoService") {
    describe("POST /todos") {
      it("should create a todo and return its id") {
        val todoService = getTodoService

        val response = makeRequest(
          todoService,
          POST(TodoPostRequestDTO("do taxes").asJson, uri"/todos")
        )

        val responseBody = getBody[TodoPostResponseDTO](response)

        response.status.code shouldEqual 201
        responseBody shouldEqual TodoPostResponseDTO(responseBody.id)
      }
    }
  }
}
```

When we run the spec:
```
INFO  🐳 [testcontainers/ryuk:0.3.4] - Creating container for image: testcontainers/ryuk:0.3.4
INFO  🐳 [testcontainers/ryuk:0.3.4] - Container testcontainers/ryuk:0.3.4 is starting: ae0bd821279180dcf1db2b740c89658c3471392d3b371d86cdefad78cc94116f
INFO  🐳 [testcontainers/ryuk:0.3.4] - Container testcontainers/ryuk:0.3.4 started in PT0.494139S
INFO  o.t.utility.RyukResourceReaper - Ryuk started - will monitor and terminate Testcontainers containers on JVM exit
INFO  o.testcontainers.DockerClientFactory - Checking the system...
INFO  o.testcontainers.DockerClientFactory - ✔︎ Docker server version should be at least 1.6.0
INFO  🐳 [mongo:5.0] - Creating container for image: mongo:5.0
INFO  🐳 [mongo:5.0] - Container mongo:5.0 is starting: f47d911dd8a8cf3a438a70f7a8b75167dc877ad8a3f3e26478afba1b9b93bb4e
INFO  🐳 [mongo:5.0] - Container mongo:5.0 started in PT0.897011S
```

We can see that a `testcontainers/ryuk` container is started in addition to our MongoDB container.
`ryuk` is responsible for cleaning up the containers we started during tests after
the tests are executed. `ryuk` needs additional privileges to run which is fine locally
but may create issues in the CI environment. That's why there is a flag that you can set up
to disable it when running in CI `TESTCONTAINERS_RYUK_DISABLED=false`

This is the basic flow of using testcontainers-scala.
Of course, there is more functionality available. For example
if you'd rather keep the image configuration in a docker file instead
of specifying it in the test spec you can do that and tell test containers
to run a docker file, or if you have docker-compose you can do the same.

As we can see testcontainer-scala provides a simple way to run containers
during tests. You can find the whole code example [here](https://github.com/jcz2/test-containers).
