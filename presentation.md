# Micro Service Orchestration

### A Functional Approach
#### Andrew Kuhnhausen - FinagleCon 2015
##### @kuhnhausen | github.com/trane | blog.errstr.com

---

# Agenda

---

## back story

---

## primitives

![100%, left](finagle_logo_small.png)
![100%, right](finch-logo.png)

^ We heard from Marius about abstracting Finagle primitives, and for those that attended the Finch workshop you saw how Vladymir and Travis took the lower level primitives from finagle-httpx and lifted them into higher-order functional structures to define a dsl for http api

---

## abstractions

^ i'm going to talk about finding some reasonable abstractions for few web components that we developed for our border of the network session composition and authentication application

---

# /me

## work

* Engineer at ![](lookout.png)
* Started Functional Programming study groups
* Diversity Committee (let's talk)
* Moving teams to work on Functional infrastructure
* We are hiring - come ride the Curry-Howard correspondence with us

---

# /me

## academia

* Static Analysis of Dalvik Bytecode for Malware Detection
* University of Utah, advisor: Matt Might

---

# /me

## why you should (not) listen

* This is my first conference talk
* I'm not an expert in Scala nor typed FP
* Lessons from building an idiomatic Scala open-source FP library
* Library is not Java friendly

---

# Some(Context)

^ It's a story as old as time

---

## ruby + rails
### ...a love story
![](love2.jpg)

^ a well intentioned group of people start using rails

---

### true luv 4ever!!1
![](love3.jpg)

^ and then they use it some more, 'cause they love it so much

---

### tests will keep us 2gether :) :) :)
![](love4.jpg)

^ and then they keep using it... until one day

---

### elho? ping? ack? ... silence
![](rails.jpg)

^ you realize that you've built a monolithic rails service that falls down under the success you've had. of course, you know the answer to this...

---

## servicesus minimus
### servii? servi?
![](unicorn.jpg)

^ it's micro services and they are great! building small, understandable components in the hope to make testing and development of new features faster and simpler is fantastic

---

## now you've just moved complexity
![](microservices.png)

^ This simplification at the service layer pushes the complexity cost to components in the service orchestration layer.

---

## orchestration
![right](orchestration.jpg)

* compose sessions
* allow for plugabble/modular authentication
* abstract away service discovery for clients

^ the goal of this talk is to tell a story about my attempt at building some hopefully useful functional abstractions on top of the excellent primitives that finagle and finch provide, in the spirit that more people will do that same thing. and that we can build a community of modern, idiomatic fp in scala libraries.

---

# context level++
![](win.jpg)

#### github.com/lookout/ngx_borderpatrol

^ our first version of this orchestration service written in nginx+lua, and it's available as an open source project, too. this talk is about building a more reusable library for the open source community that we could use ourselves to build our edge-of-the-network orchestrator

---

# Library Design

^ i wanted to talk a bit about making flexibile apis for the users of your library, and how functional programming gives you a really reusable approach

---

## why inheritance isn't good enough

```scala
trait Mammal {
  // implement mammal
}

case object Human extends Mammal
case object Cat extends Mammal
```

---

## inheritance for end users

### define a *general* type and silently accept more *specific* types (subtypes)

^ this is a very inflexible model that puts the onus on the library writter to encapsulate all potential uses.

----

### shoe store

![fit](catshoes.jpg)

```scala
trait ShoeStore {
  def get(m: Mammal): Kicks
}
```

---

### others want our shoes

```scala
case object Bird extends Animal
case object Lizard extends Reptile
case object Velociraptor extends Bird with Lizard

:(
```
![inline](veloci.jpg)

---

## interfaces

### define a *specific* description and silently allow more *general* types

---

## icanhazshoes?

* Anything with `Feet` can (probably) wear shoes
* Support `Reptile`, `Bird`, `Mamamal`, but not `ImperialMetrics`

---

### adapter pattern

---

### adapter pattern

interface membership is determined at *definition*

^ so it's cumbersome as a library user, and doesn't work on types you don't define, e.g. String

---

## monkey patching

^ in ruby, we could just monkey patch - which gives us the more ad-hoc at-use membership that interfaces don't give us. unfortunately, only mean people use monkey patching.

---

## type classes

^ In Functional Programming, we have the best of both worlds: program to an interface and - the spirit in which monkey-patching happens - ad-hoc polymorphism.

---

# Abstracting

^ let's talk about a way we can model Sessions and utilize type classes to enable arbitrary data to be stored

---

## web sessions

![left](cookie.jpg)

```scala
trait Session[A] {
  val id: SessionId
  val data: A
}

trait SessionId {
  val expiry: Time
  val entropy: Seq[Byte]
  val secret: Secret
  val signature: Seq[Byte]
}
```
^ a session is a simple container, a unique cryptographically signed id that is given as the value to a cookie, and some data the library user wants to store along with this session. 

---

## interface

```scala
type K
type Get[B]: K => Store => Option[B], 
type Put[B]: K => B => Store => Unit
```
^ a simple definition of a store would be to take some key and some value and a store

---

## A !=? B

^ but, the problem is fitting the data in a session into that store

---

## laws

* `A => B`
* `B => Option[A]`

^ What we need is some function that can convert the `data` a user of our library wants to store into the type that our backend stores: `A => B`. We also need some way to convert from what was stored into what the user expects: `B => Option[A]`.

---

### define the general law

```scala
trait Encoder[A, B] {
  def encode(a: A): B
  def decode(b: B): Option[A]
}
```

---

### define a less general law

^ Finagle already has a data type that is used throughout their various protocol implementations: `Buf`. This type is a nice wrapper for a `Array[Byte]` with many helper methods for types like `String`, `Utf8`, `U32`, etc.

---

### Buf

```scala
type EncodeSession[A] = Encoder[A, Buf]

// or

trait EncodeSession[A] {
  def encode(data: A): Buf
  def decode(buf: Buf): Option[A]
}

```

^ So, knowing that we are building on top of Finagle, and we are dealing with `Session[A]`, we can simplify that trait to require Buf. This opinionated part of the design is a trade-off for ease of use over flexibility.

---

## defaults

![right,fit](borderpatrol.png)

^ One of the main use cases is to save the location that an unauthenticated user tried to access so that we can send them their directly after logging in. So, we need an implementation for the initial `Request`

---

## EncodeSessionRequest

```scala
implicit object EncodedSessionRequest 
    extends EncodeSession[httpx.Request] {
  
  def encode(data: httpx.Request): Buf = 
    Buf.ByteArray.Owned(data.encodeBytes())
  
  def decode(buf: Buf): Option[httpx.Request] = 
    Try { Request.decodeBytes(Buf.ByteArray.Owned.extract(buf)) }.toOption
}
```

^ This is another really nice example of utilizing Request and Buf. The first iteration of this that I did, I encoded it into Json and then to a byte array.

---

## use the type class

```scala
trait SessionStore {
  def put[A : EncodeSession](session: Session[A]): Future[Unit]
  def get[A : EncodeSession](id: SessionId): Future[Option[Session[A]]]
}
```

---

### a simple memcached implementation

```scala
case class MemcachedSessionStore(store: memcachedx.BaseClient[Buf]) 
    extends SessionStore {
  def put[A : EncodeSession](session: Session[A]): Future[Unit] =
  	 store.set(session.id, 0, 60, implicitly[EncodeSession[A]](session.data))
  	 
  def get[A : EncodeSession](id: SessionId): Future[Option[Session[A]]]
     store.get(id).map {opt =>
       opt.flatMap {buf => 
         Session(id, implicitly[EncodeSession[A]].unapply(buf))
       }
     }
}
```

^ how easy would it be to implement a Redis backend or any other Backend? This is the power from Finagle.

---

## what just happened?

^ Now, `Session` and `SessionStore` only care about a contract of two functions to work with any type of data that a user wants to cache with their session. Users can implement this functionality for any type, even those they don't define!

---

## more advanced fun-time

### Session[A] + Functor[F[_]] + Free Monad
#### :) yesplz! (:

^ `Session[A]` can be lifted into a `Functor[Session[_]]`
Once we have a functor, we can define an interpreter for our database operations using the Free Monad
Now `get` and `put` become actual types of `Get` and `Put` and we can know even more about the behavior at compile-time
Feel free to implement this for fun, we are accepting pull requests

---

## auth as a type class

^ we also have auth type classes to - hopefully - be able to plug in different authentication mechanisms for our endpoints

---

# Primitives

^ Knowing how easy it is to implement any backend store for our Session interface, let's go a bit deeper into the primitives that both Finagle and Finch give us to see why

---

## Finagle
### Service 

```scala
type Service[Req, Rep] = Req => Future[Rep]
```

### Filter

```scala
type Filter[ReqIn, RepOut, ReqOut, RepIn] =
  (Req, Service[ReqOut, RepIn]) => Future[RepOut]
```

^ these types boil down to simple partial functions, and applying a filter to a service yields a service. this works really simply for RPC like Thrift or Protobufs. The problem comes when you have to deal with HTTP, where the protocol is so general that you end up with a lot of messy code that can't be checked at compile time.

---

## Finch

### Router / RequestReader / ResponseBuilder
```scala
type Router[A] = Request => Option[A]
```

```scala
type RequestReader[A] = Request => Future[A]
```

```scala
type ResponseBuilder[A] = (A, EncodeResponse[A]) => Response
```
^ finch lifts the request type in some higher-order functions that allow us to encode those possibilities into value types that compose together. this is the level that we want to be working at for HTTP

---

# Composability

^ Finch takes an scala idiomatic approach to a functional library, this is where I learned a lot about defining types in a general and composable way.

---

## Router Combinator

```scala
type Router[A] = Request => Option[A]
```

^ in a low-level, routers can be treated as a simple function from a Request to an Option of some generic type. this means we get to have meaningful types instead of having to go all the way to an http Request.

---

## Combinator DSL

```scala
// GET /users/:id/tickets/:id
get("users" / int("userId") / "tickets" / int("ticketId"))
```

---

## Using these primitives

^ in bp, we have the notion of which service a user wants to talk to baked in to the path

---

### Router for Path based Service
`http://example.com/:service/...`

---

### Router for Subdomain based default Service
`http://service.example.com/`

---

## Path based Service

#### lookout.com/:service/ => :service

```scala
implicit val pathMap = Map(("ent" -> "enterprise"), ("my" -> "consumer"))

def validatePath(path: String)(implicit services: Map[String, String]): Option[String] = 
  services.get(path)

object servicePath extends Extractor("service", validate(_))
```

---

## Default subdomain based

#### <sub>.lookout.com/ => sub

```scala
case class DefaultServiceDomain(g: String => Option[String])
    extends Extractor[String]("defaultService", identity) {

  import Router._

  override def apply(input: Input): Option[(Input, () => Future[String])] =
    for {
      h <- input.request.host
      s <- g(h)
    } yield (input.drop(1), () => Future.value(s))
}

val domainMap = Map(("www" -> "consumer"), ("enterprise" -> "enterprise"))
val subdomain = DefaultServiceDomain(domainMap.get(_))
```

---

## Compose them

```scala
val serviceRouter = servicePath | subdomain
```

```
  if (:service in path) then:
    mapping(:service)
  if (:service in subdomain) then:
    default(:service)
  else
    halt with 400
  end
```
^ we've effectively encoded the sequential conditional logic into values and pushed the condition implementation down into composable boxes!

---

## w00t! w00t!

---

## What this looks like

```scala
val endpoints = (
  (Get / serviceRouter /> AuthenticatedEndpoint) :+:
  (Post / "login" /> LoginService))
)

val server = Httpx.serve("localhost:8080", endpoints.toService)
```

^ using coproduct routers, we can define both authenticated and unauthenticated endpoints

---

# Things bp does
* Sessions
* Authentication (Basic, internal)
* Encryption for data at rest

---

# Things bp is about to do
* CSRF via double submit cookie
* Periodic secret key generation

---

# Things you can help with
* github.com/lookout/borderpatrol/issues
* Build your own abstractions on Finagle/Finch

---

# Thank you

### @kuhnhausen | github.com/trane | blog.errstr.com

---

