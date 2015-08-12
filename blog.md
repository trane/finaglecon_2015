# Micro services orchestration - a functional approach

As we get close to [FinagleCon](https://finagle.github.io/finaglecon/), I decided to write a blog post to accompany the [talk I'll be giving](http://sched.co/3v0n).

## Introduction

A micro services architecture encourages building small, understandable components in the hope to make testing and development of new features faster and simpler. This simplification at the service layer pushes the complexity cost to components in the service orchestration layer like authentication, routing, and sessions.

In this post, I will highlight a functional, type-safe, and generic programming approach to authentication and sessions for micro services built on top of [Finagle](https://twitter.github.io/finagle/) and [Finch](https://github.com/finagle/finch/); focus will be on the modularity and extensibility gained by composing Finagle’s primitives with Scala’s type classes, while going through the design of a functional library.

## Baby's first steps

About one year ago, [Rob Wygand](https://github.com/rwygand) introduced our open-source border-of-the-network authentication and session management project: [Border Patrol](http://hackers.lookout.com/2014/06/introducing-borderpatrol/). It was our first go at tackling the complexities that arose in our infrastructure as we moved to a service oriented architecture. While it's not required reading for this post, I recommend reading that article for a more detailed explanation of the concepts and architectural reasoning behind the project.

### Lessons

As an open source project, [Border Patrol](https://github.com/lookout/ngx_borderpatrol) worked - but, only for our specific implementation. Namely, both our token-based authentication and credential systems were required to make it work (neither of which are open source). Further, we found that deploying our internal implementation into various environments meant an unhappy coupling of business logic living in both the app and in configuration management.

### Hackathon

Shortly before I came on board to the project, we had one of our [3-day hackathons](https://www.lookout.com/about/careers/engineering) at Lookout. A few of us decided to investigate what it would be like to implement Border Patrol on top of [Twitter's Finagle](https://twitter.github.io/finagle/). We got it working - for the most part - despite the short time-frame. However, it wasn't any less complex or messy than the Nginx+Lua implementation. But, it was clear that Finagle gave us the primitive building blocks to enable us to deliver on the promise of a *reusable* open-source project.


## Designing for reusability

We needed a way to decouple the business logic from our specific implementation. The list of things that we want to decouple:

* Authentication - support *any* arbitrary HTTP compatible authentication
* Backend stores - this is where Finagle is very helpful
* Persistence/Expiry - should be handled easily via configuration
* Encryption - support data at rest encryption with configurable strength
 
### In* all the things!

Inheritance is a notion in many programming languages that boils down to defining a general description of your type and silently accepting subtypes, e.g. Human and Cat are both sub-types of Mammal. So, our new `ShoeStoreDb` can outfit both my furry friend and me with a new set of kicks: `get(m: Mammal): Kicks`. Unfortunately, that leaves out my friends' birds and lizards, and their friends' velociraptors.

Perhaps, instead, we should ask that anything with `Feet` can get the latest in cowboy footwear - this is an interface. Interfaces boil down to defining the most specific description of your type and silently accepting more general types.

One common OO-pattern for decoupling specific implementation from business logic is the [Adapter Pattern](https://en.wikipedia.org/wiki/Adapter_pattern) - make everything an interface. This seems like such a great option, except for one pitfall: interface membership is determined at definition. So, adding functionality to existing interfaces that we work with or our potential users would work with is very cumbersome. It's not like in Ruby where you can monkey-patch and forget! (not recommended - not to be confused with Patches the Monkey who needs a new pair of stilletos)

In Functional Programming, we have the best of both worlds: program to an interface and - the spirit in which monkey-patching happens - ad-hoc polymorphism.

### Type Classes

At the surface, Type Classes are just interfaces. But, they have one key difference: membership is determined ad-hoc, at usage. This means that you can make an interface for types that you didn't define nor have access to. Watch out for the new fall manatee hiking boot lineup!

Utilizing Type Classes, we can define a set of laws that members of the Type Class must abide by. So long as the user provides proof that they follow those laws, we will accept that type.
