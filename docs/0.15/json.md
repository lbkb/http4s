---
layout: default
title: JSON handling
---

## Pick a library

### Argonaut

Argonaut-shapeless for automatic codec derivation.

Note: argonaut-shapeless is not yet available for argonaut-6.2.

```scala
libraryDependencies += Seq(
  "org.http4s" %% "http4s-argonaut" % "0.15.0a-SNAPSHOT",
  "com.github.alexarchambault" %% "argonaut-shapeless_6.2" % "1.2.0"
)
```

### Circe

Circe-generic for automatic codec derivation.

```scala
libraryDependencies ++= Seq(
  "org.http4s" %% "http4s-circe" % "0.15.0a-SNAPSHOT",
  "io.circe" %% "circe-generic" % "0.4.1"
)
```

### Json4s

Json4s supports two backends.  Choose one of:

```scala
libraryDependencies += "org.http4s" %% "http4s-json4s-native" % "0.15.0a-SNAPSHOT"
libraryDependencies += "org.http4s" %% "http4s-json4s-jackson" % "0.15.0a-SNAPSHOT"
```

## Import it

The import statement is one of the following:

```scala
import org.http4s.argonaut._
import org.http4s.circe._
import org.http4s.json4s.jackson._
import org.http4s.json4s.native._
```

## Deriving codecs

To use the functions provided by http4s, the corresponding implicit instances
need to be in scope. For argonaut, there is [argonaut-shapeless], circe has the
[circe-generic] module. With json4s, there's [jsonExtract], with the
corresponding drawbacks.

## Sending JSON

The usage is the same for client and server, both points use an
`EntityEncoder[T]` to transform the outgoing data into a scala class.
One of the imports above brings into scope an `EntityDecoder[Json]` 
(or `EntityDecoder[JValue]` in the case of json4s).

In circe, when one has an `Encoder` instance, `.asJson` can be
called to get to a `Json`.  In the example below, we convert the
`Hello` case class to JSON for rendering in an `Ok` response.

```scala
import io.circe._
// import io.circe._

import io.circe.generic.auto._
// import io.circe.generic.auto._

import io.circe.syntax._
// import io.circe.syntax._

import org.http4s._
// import org.http4s._

import org.http4s.circe._
// import org.http4s.circe._

import org.http4s.dsl._
// import org.http4s.dsl._

case class User(name: String)
// defined class User

case class Hello(greeting: String)
// defined class Hello

val jsonService = HttpService {
  case r @ POST -> Root / "hello" =>
    r.as(jsonOf[User]).flatMap(user =>
      Ok(Hello(s"Hello, ${user.name}").asJson)
    )
}
// jsonService: org.http4s.HttpService = Kleisli(<function1>)

import org.http4s.server.blaze._
// import org.http4s.server.blaze._

val builder = BlazeBuilder.bindHttp(8080, "localhost").mountService(jsonService, "/")
// builder: org.http4s.server.blaze.BlazeBuilder = org.http4s.server.blaze.BlazeBuilder@5831cddf

val blazeServer = builder.run
// blazeServer: org.http4s.server.Server = BlazeServer(/127.0.0.1:8080)
```

## Receiving JSON

The `EntityDecoder[T]` instances work similarly to the `EntityEncoder[T]`
instances, except it's `jsonOf` to create the instances.

Let's talk to the server defined above.

```scala
import org.http4s.client.blaze._
// import org.http4s.client.blaze._

import org.http4s.Uri
// import org.http4s.Uri

val httpClient = PooledHttp1Client()
// httpClient: org.http4s.client.Client = Client(Kleisli(<function1>),scalaz.concurrent.Task@3ddb34a9)

val req = Request(uri = Uri.uri("http://localhost:8080/hello"), method = Method.POST).withBody(User("Anabelle"))(jsonEncoderOf)
// req: scalaz.concurrent.Task[org.http4s.Request] = scalaz.concurrent.Task@295f48ae

httpClient.expect(req)(jsonOf[Hello]).run
// <console>:44: warning: method run in class Task is deprecated: use unsafePerformSync
//        httpClient.expect(req)(jsonOf[Hello]).run
//                                              ^
// res0: Hello = Hello(Hello, Anabelle)
```

And clean everything up.

```scala
httpClient.shutdownNow()

blazeServer.shutdownNow()
```

## Talking to the Github API

As an example, we'll use the [github-orgs] endpoint. The data is shortened a bit
for clarity.

```json
[
  {
    "id": 3692188,
    "name": "http4s",
    "full_name": "http4s/http4s",
    "owner": {
      "login": "http4s",
      "id": 1527492
      // ...
    },
    "private": false,
    "html_url": "https://github.com/http4s/http4s",
    "description": "A minimal, idiomatic Scala interface for HTTP",
    "size": 21365,
    "stargazers_count": 533,
    "language": "Scala",
    "forks_count": 118,
    "open_issues_count": 69,
    "forks": 118,
    "open_issues": 69,
    "watchers": 533
    // ...
  }
  // ...
]
```

To capture this structure, we'll need corresponding case classes.

```scala
case class User(login: String, id: Long)
// defined class User

case class Repo(id: Long, name: String, full_name: String, owner: User, `private`: Boolean, html_url: String, description: String, size: Int, stargazers_count: Int, language: String, forks_count: Int, open_issues_count: Int, forks: Int, open_issues: Int, watchers: Int)
// defined class Repo
```

This parts skips over the [client] explanation. We'll use circe, with
[circe-generic] for codec derivation. The JSON decoder is provided by
circe-generic, and passed to the `client.expect` method via the second
argument, which is usually an implicit `EntityDecoder`.  In this case,
we want to be more explicit.

<!-- For more information about the uri templating, visit [uri]. -->

```scala
import scalaz.concurrent.Task
// import scalaz.concurrent.Task

import org.http4s.util.string._
// import org.http4s.util.string._

val httpClient = PooledHttp1Client()
// httpClient: org.http4s.client.Client = Client(Kleisli(<function1>),scalaz.concurrent.Task@423503b8)

def repos(organization: String): Task[List[Repo]] = {
  val uri = Uri.uri("https://api.github.com/orgs") / organization / "repos"
  httpClient.expect(uri)(jsonOf[List[Repo]])
}
// repos: (organization: String)scalaz.concurrent.Task[List[Repo]]

val http4s = repos("http4s")
// http4s: scalaz.concurrent.Task[List[Repo]] = scalaz.concurrent.Task@60be427c

http4s.map(_.map(_.stargazers_count).mkString("\n")).run
// <console>:49: warning: method run in class Task is deprecated: use unsafePerformSync
//        http4s.map(_.map(_.stargazers_count).mkString("\n")).run
//                                                             ^
// res3: String =
// 614
// 88
// 8
// 71
// 1
// 5
// 0
// 0

httpClient.shutdownNow()
```


[argonaut-shapeless]: https://github.com/alexarchambault/argonaut-shapeless
[circe-generic]: https://github.com/travisbrown/circe#codec-derivation
[jsonExtract]: https://github.com/http4s/http4s/blob/master/json4s/src/main/scala/org/http4s/json4s/Json4sInstances.scala#L29
[client]: client.html
[github-orgs]: https://developer.github.com/v3/repos/#list-organization-repositories
[uri]: url.html