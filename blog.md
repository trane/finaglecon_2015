# Micro services orchestration - a functional approach

As we get close to [FinagleCon](https://finagle.github.io/finaglecon/), I decided to write a blog post to accompany the [talk I'll be giving](http://sched.co/3v0n).

## Introduction

A micro services architecture encourages building small, understandable components in the hope to make testing and development of new features faster and simpler. This simplification at the service layer pushes the complexity cost to components in the service orchestration layer like authentication, routing, and sessions.

In this post, I will highlight a functional, type-safe, and generic programming approach to authentication and sessions for micro services built on top of [Finagle](https://twitter.github.io/finagle/) and [Finch](https://github.com/finagle/finch/); focus will be on the modularity and extensibility gained by composing Finagle’s primitives with Scala’s type classes, while going through the design of a functional library.

## Baby's First Steps

About one year ago, [Rob Wygand](https://github.com/rwygand) introduced our open-source border-of-the-network authentication and session management project: [Border Patrol](http://hackers.lookout.com/2014/06/introducing-borderpatrol/). It was our first go at tackling the complexities that arose in our infrastructure as we moved to a service oriented architecture. While it's not required reading for this post, I recommend reading that article for a more detailed explanation of the concepts and architectural reasoning behind the project.

### Lessons

As an open source project, [Border Patrol](https://github.com/lookout/ngx_borderpatrol) worked - but, only for our specific implementation. Namely, both our token-based authentication and credential systems were required to make it work (neither of which are open source). Further, we found that deploying our internal implementation into various environments meant an unhappy coupling of business logic living in both the app and in configuration management.

### Hackathon

Shortly before I came on board to the project, we had one of our [3-day hackathons](https://www.lookout.com/about/careers/engineering) at Lookout. A few of us decided to investigate what it would be like to implement Border Patrol on top of [Twitter's Finagle](https://twitter.github.io/finagle/). We got it working - for the most part - despite the short time-frame. However, it wasn't any less complex or messy than the Nginx+Lua implementation. But, it was clear that Finagle gave us the primitive building blocks to enable us to deliver on the promise of a *reusable* open-source project.


## Decouple to Enable

A good library is one which is opinionated in the right ways

### Pipes

### Type Classes
Adapter pattern
