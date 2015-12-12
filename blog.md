# Micro services orchestration - a functional approach

I decided to write a blog post to accompany the [talk I gave](http://sched.co/3v0n) at FinagleCon 2015.

## Introduction

A micro services architecture encourages building small, understandable
components in the hope to make testing and development of new features faster
and simpler. This simplification at the service layer pushes the complexity
cost to components in the service orchestration layer like authentication,
routing, and sessions.

In this post, I will highlight one of the functional, type-safe, and generic
programming approaches to sessions for micro services built on top of
[Finagle](https://twitter.github.io/finagle/) and
[Finch](https://github.com/finagle/finch/); focus will be on the modularity and
extensibility gained by composing Finagle’s primitives with Scala’s type
classes, while going through the design of a functional library.

## Baby's first steps

About one year ago, [Rob Wygand](https://github.com/rwygand) introduced our
open-source border-of-the-network authentication and session management
project: [Border
Patrol](http://hackers.lookout.com/2014/06/introducing-borderpatrol/). It was
our first go at tackling the complexities that arose in our infrastructure as
we moved to a service oriented architecture. While it's not required reading
for this post, I recommend reading that article for a more detailed explanation
of the concepts and architectural reasoning behind the project.

### Lessons

As an open source project, [Border
Patrol](https://github.com/lookout/ngx_borderpatrol) worked - but, only for our
specific implementation. Namely, both our token-based authentication and
credential systems were required to make it work (neither of which are open
source). Further, we found that deploying our internal implementation into
various environments meant an unhappy coupling of business logic living in both
the app and in configuration management.

### Hackathon

Shortly before I came on board to the project, we had one of our [3-day
hackathons](https://www.lookout.com/about/careers/engineering) at Lookout. A
few of us decided to investigate what it would be like to implement Border
Patrol on top of [Twitter's Finagle](https://twitter.github.io/finagle/). We
got it working - for the most part - despite the short time-frame. However, it
wasn't any less complex or messy than the Nginx+Lua implementation. But, it was
clear that Finagle gave us the primitive building blocks to enable us to
deliver on the promise of a *reusable* open-source project.

## Designing for reusability

There are a few things that we're going to count on in designing this library
that are particularly well suited for a language like Scala:

* Modularity (roots from Standard ML, Modula-2, Modula-3)
* Interfaces (á-la an ML derived languages: Haskell, OCaml)

Most importantly, we want to be able to specify the functionality of a
component, abstract usable programs over that specification, and provide
instances of that program that are directly usable and extensible for users of
that component.

### Modulum oblongatum

One of the issues with the original Border Patrol `ngx_borderpatrol` was the
lack of separation of abstraction and implementation. To aide in a more strict
decoupling of components, we need to distill what Border Patrol does into its
most simple modules:

* Authentication - support *any* arbitrary HTTP compatible authentication
* Web Sessions - a serialization and persistence layer
* Encryption - encryption for data at rest and other crypto primitives
* Server - this is the program that composes the other modules

This informs both the maintainers and users of our library about the modular
hierarchy of behavior.  But, also importantly, it more effectively structures
our code to scope our implicit behavior to clear explicit boundaries -
essentially getting the best of both worlds from Haskell's global implicit
behavior and OCaml's modular explicit behavior. For more on the deeper meaning
of this, I recommend reading [Modular Type
Classes](https://www.cse.unsw.edu.au/~chak/papers/mtc-popl.pdf)

### Interface primer

Let's get silly.

For the of an example, suppose we are designing a component for a high-end pet
store that would search a database for a list of fancy shoes for cats and dogs:
`ShoeStoreDb`

Inheritance is a notion in many programming
languages that boils down to
defining a general description of your type and silently accepting subtypes,
e.g. Dog and Cat are both sub-types of Mammal. So, our new `ShoeStoreDb` can
outfit both my furry friend and me with a new set of kicks: `get(m: Mammal):
Kicks`. Unfortunately, that leaves out my friends' birds and lizards, and their
friends' velociraptors.

Perhaps, instead, we should ask that anything with `Feet` can get the latest in
cowboy footwear - this is an interface. Interfaces boil down to defining the
most specific description of your type and silently accepting more general
types.

One common OO-pattern for decoupling specific implementation from business
logic is the [Adapter Pattern](https://en.wikipedia.org/wiki/Adapter_pattern) -
make everything an interface. This seems like such a great option, except for
one pitfall: interface membership is determined at definition. So, adding
functionality to existing interfaces that we work with or our potential users
would work with is very cumbersome. It's not like in Ruby where you can
monkey-patch and forget! (not recommended - not to be confused with Patches the
Monkey who needs a new pair of stilletos)

In Functional Programming, we have the best of both worlds: program to an
interface and - the spirit in which monkey-patching happens - ad-hoc
polymorphism.

### Type Classes

At the surface, Type Classes are just interfaces. But, they have one key
difference: membership is determined ad-hoc, at usage. This means that you can
make an interface for types that you didn't define nor have access to, e.g.
`String`. Watch out for the new fall manatee hiking boot lineup!


Utilizing Type Classes, we can define a set of laws that members of the Type
Class must abide by. So long as the user provides proof that they follow those
laws, we will accept that type. This is how we can support data that users of
our library might want to store in their session.

#### Rules for Sessions to live by

A session is defined as:

```scala
type SessionId = String

trait Session[A] {
  id: SessionId
  data: A
}
```

What we need is some function that can convert the `data` a user of our library
wants to store into the type that our backend stores: `A => B`. We also need
some way to convert from what was stored into what the user expects: `B =>
Option[A]`.

Let's define these rules as a `trait`:

```scala
trait Encoder[A, B] {
  def apply(s: A): B
  def unapply(b: B): Option[A]
}
```

Finagle already has a data type that is used throughout their various protocol
implementations: `Buf`. This type is a nice wrapper for a `Array[Byte]` with
many helper methods for types like `String`, `Utf8`, `U32`, etc. So, knowing
that we are building on top of Finagle, and we are dealing with `Session[A]`,
we can simplify that trait to:

```scala
trait EncodeSession[A] {
  def apply(data: A): Buf
  def unapply(buf: Buf): Option[A]
}
```

#### Provide default implementations

We already know that part of Border Patrol's functionality is to save the
location that an unauthenticated user tried to access so that we can send them
their directly after logging in. So, we need an implementation for the initial
`Request`:

```scala
implicit object EncodedSessionRequest extends EncodeSession[httpx.Request] {
  def apply(data: httpx.Request): Buf =
    Buf.ByteArray.Owned(data.encodeBytes())
  def unapply(buf: Buf): Option[httpx.Request] =
    Try { Request.decodeBytes(Buf.ByteArray.Owned.extract(buf)) }.toOption
}
```

#### Code with Type Classes

Now we can define the interface for our store to allow for any type of session
data to be stored.

```scala
trait SessionStore {
  def put[A : EncodeSession](session: Session[A]): Future[Unit]
  def get[A : EncodeSession](id: SessionId): Future[Option[Session[A]]]
}
```

Implementing this for Memcached is straight forward now

```scala
case class MemcachedSessionStore(store: memcachedx.BaseClient[Buf])
    extends SessionStore {

  def put[A : EncodeSession](session: Session[A]): Future[Unit] =
  	 store.set(session.id, 0, 60, implicitly[EncodeSession](session.data))

  def get[A : EncodeSession](id: SessionId): Future[Option[Session[A]]]
     store.get(id).map(_.flatMap(buf =>
       Session(id, implicitly[EncodeSession].unapply(buf)))
}
```

#### What just happened?

Well, we just allowed for more general types to be defined by users of our
library and described a set of rules (`A => B` and `B => Option[A]`) that those
types must follow. Then we provided a default implementation that users can
import into their code and get for free!

## Encode our logic in types
