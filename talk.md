
Type Classes are just interfaces with one key difference:

interface membership is determined at definition
type class membership is determined ad-hoc, whenever you provide proof of membership
... because type classes are implemented using traits and implicits, and implicits are scoped,
this means that you can provide alternative proof of membership

interfaces are very useful:
  - write the most general definition and silently accept subtypes
  - write the most specific definition and silently accept more general types

Write containers that adapt easily to even higher-kinded types:
  - Session[A] looks really familiar when we wrap it in a Functor[F[_]], doesn't it?
  - If we can proof a functor for Session[A], we can utilize the Free Monad + Interpreter = DB

Finagle provides well thought-out primitives for RPC:
  - type Service[Req, Rep] = Req => Future[Rep]
  - type Filter[ReqIn, RepOut, ReqOut, RepIn] = (Req, Service[ReqOut, RepIn]) => Future[RepOut]

Finch abstracts over Finagle Httpx with:
  - type Router[A] = Request => Option[A]
  - type RequestReader[A] = Request => Future[A] (with flatMap, map, and some other combinators)
  - type ResponseBuilder[A] = (A, EncodeResponse[A]) => Response

How can we use these building blocks to build more abstractions?

Web Sessions

Session is some state held on the server for a client, where the client presents an identifier and the server can hydrate that state

type Session[A] = Id => Session[A]
type Store[A] = get: Id => Option[A], put: (Id, A) => Unit

Make the store polymorphic to the type it stores:

type Store = get[A]: Id => Option[A], put[A]: (Id, A) => Unit

Further, we are working with the web so, we want to wrap these in a Future

trait Store[Id] {
  def get[A](id: Id): Future[Option[A]]
  def put[A](id: Id, a: A): Future[Unit]
}

case object InMemoryStore extends Store[String] {
  var store: mutable.Set[(String, ???)] = ???
}
Wait? What if

trait SessionStore {

}

EncodeResponse[A] { A => Buf }

