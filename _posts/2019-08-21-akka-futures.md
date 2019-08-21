---
layout: post
title: "Akka and Futures: piping and asking"
description: "An in-depth look on how Futures integrate with Akka actors"
comments: true
keywords: "akka, actors, actor model, future, ask, pipe"
---

Performing asynchronous operations through the use of `Future`'s can be a powerful tool when combined with Akka actors.
There are, however, some pitfalls which you should be aware of in order not to cut yourself.
In this post I'll give description of what `Future`s are, how they integrate with actors and what common patterns exist to do so.

# What are Futures?

In scala, `Future`'s are a data structure which contain a value for which the computation may take time.
The computation required to get this value will be executed concurrently on another thread than that of the current program.
We can then define blocking or non-blocking computations based on the value that results form the computation.
A common place where we see such time-consuming calculations are database operations.
If we want to retrieve a user from a database we know we will not get this user back instantly.
We can model this by putting the user we will evetually get back in a `Future`.
This allows our program to do other things while the database is running, thus not wasting unnecessary time.
If you want to get the data from the `Future` and use it in the thread of your program,
you are going to have to wait/block at some point to ensure that the data is there to be used.
By delaying this blocking as long as possible, however, we can have multiple pieces of computation running at the same time,
thus speeding up our program.

```scala
import scala.concurrent.Await

val userFuture: Future[User] = userDAO.getUser(userID)

// this will be executed in another thread when the user is retrieved!

val userNameFuture = userFuture.map(_.name) 

// our program can do other things here while the user is being retrieved

...

// we want to access the username here, so we have to block!

val userName = Await.result(userNameFuture, 10 seconds)

println(userName)
```

Always try to avoid blocking if possible! The above example could be rewritten to the following asynchronous equivalent:

```scala
userName.onComplete {
  case Success(name) => println(name)
  case Failure(e) => throw e
}
```

The above example illustrates another property of the computation that a `Future` represents:
the computation not only takes time, it might also fail!
In the database example, the connection might be closed, resulting in an exception being thrown in the thread of the `Future`.
When a `Future[T]` completes, it contains a `Try[T]`, which you can think of as an `Either[Exception, T]`.
The most notable difference is that `Either` is a `Left` or `Right` while a `Try` is a `Success` or `Failure`.

## Execution Contexts

So far I've talked about the computation inside a future happening in "another" thread.
Where is this thread and do we have any form of control over it you might ask.
The answer is: yes. When we create a future, we need to pass a value of type `ExecutionContext` implicitly to the `Future`.
The `ExecutionContext` provides a thread to execute the calculation on.
These contexts can amongst others spawn a new thread or pick one from some thread pool, depending on the implementation.
I'll not go into depth on this topic here, for more reading please see the official scala docs [[^1]]

# Actors and futures

Although `Future`'s are part of the stdlib of scala and actors are not, they intuitively feel like a good match.
Both provide handles to design concurrent applications with relative ease.
Doing computations inside your application concurrently through actors complements concurrent IO operations with `Future`'s nicely.
There are some dangers in this combination though which are not directly obvious.

The Akka docs warn:
> When using future callbacks, such as onComplete, or mapsuch as thenRun, or thenApply inside actors you need to carefully avoid closing over the containing actor’s reference, 
i.e. do not call methods or access mutable state on the enclosing actor from within the callback. 
This would break the actor encapsulation and may introduce synchronization bugs and race conditions because the callback will be scheduled concurrently to the enclosing actor. 
Unfortunately there is not yet a way to detect these illegal accesses at compile time. [[^2]]

An example of an unsafe way to use a `Future` in an acter can be seen below.

```scala
class UnsafeActor(userDAO: UserDAO) extends Actor {
  import UnsafeActor._
  
  // all actors have their own ExecutionContext!
  
  implicit val ec: ExecutionContext = context.dispatcher
  
  def receive = {
    case GetUser(userId) => userDAO.getUser(userId).onComplete {
      case Success(user) => context.sender() ! GetUserResult(user)
      case Failure(e) => throw e
    }
  }
}

object UnsafeActor {
  case class GetUser(id: Int)
  case class GetUserResult(user: User)
}
```

What is going wrong here?
At first glance everything seems sensible, we get a request for a user, retrieve it and send it back to who requested it
(this is stored in `context.sender`).
However, the `onComplete()` operation is non-blocking (which in itself is a good thing).
This makes that as we start retrieval of the user and define what to do with it once we get it, we send it to another thread and continue doing other things inside the actor.
Since this was the last thing we defined to do with the incoming message, we move on to the next message.
If the user is retrieved while we've moved on to the next message, `context.sender` will have changed.
This means that we will send the found user to the wrong actor!
You can probably see how this can produce incredibly nasty bugs which would take many hours to fix if you are not aware of this.

# The pipe pattern

So how should we handle the above example then?
You can choose to be extremely careful not to mess with any mutable state within `onComplete` which would make everything safe.
However, a mistake made by you or your coworker here will easily slip through during even the most scrutinising code reviews.
Instead, use Akka's `pipe` pattern.

The pipe pattern allows to have the content of a `Future` sent to the mailbox of an actor when it completes.
By blending the future in with all other messages being sent, it will become much more easy to reason about your code.
A solution to the above example using piping looks like:

```scala
class SafeActor(userDAO: UserDAO) extends Actor {
  import SafeActor._
  
  // all actors have their own ExecutionContext!
  
  implicit val ec: ExecutionContext = context.dispatcher
  
  def receive = {
    case GetUser(userId) => 
      userDAO.getUser(userId).pipTo(self)
      context.become(waitingForUser(context.sender))
  }
  
  def waitingForUser(originalSender: ActorRef) = {
    case user: User =>
      originalsender ! GetUserResult(user)
      context.become(receive) 
  }  
}

object SafeActor {
  case class GetUser(id: Int)
  case class GetUserResult(user: User)
}
```

As you can see, the sender is now passed around as a value and will not change.
When the user is retrieved we can safely use `originalSender` to send the user back to whoever requested it.
The reason to become `waitingForUser` where we do not handle new `GetUser` messages is that this
could result in multiple users being retrieved at the same time.
We then would not know which one completes first and thus where to send that result to!

A detail on the pipe pattern: when the `Future` completes successfully and we get a `Success(x)`,
it will be unwrapped and `x` will be sent to who we piped to.
If the `Future` fails, however, the resulting `Failure` will **not** be unwrapped and a `Future(e)` will end up in the mailbox!

# The ask pattern

The `ask` pattern is another way in which futures often occur in actor systems.
It is a pattern which blends well together with the `pipe` pattern.
Asking is an alternative to regular message passing.
the usual way to send a message to an actor is by telling, denoted by the `!` operator.
This is fire-and-forget and if the receiver responds with another message there will be no direct way to link this response back to the initial message.
For most cases this should be fine, but there are cases where it is not.
In those cases `ask` can be of help, denoted by the `?` operator.
To utilise this we need to have an `ExecutionContext` and a `Timeout`. The `Timeout` contains a duration after which the produced `Future` will be failed with a `AskTimeoutException`

```scala
import akka.pattern.ask

implicit val ec = ExecutionContext.global
implicit val timeout = Timeout(2 seconds)

val f = someActor ? "Hi"
```

We are now left with `f` which we know will eventually complete. 
The actor which was asked can complete this future by sending a message to the `sender` of the received message.
When you ask something of an actor, an intermediate actor is created which will send the message to the actor you ask something of.
This intermediate actor handles the completing of the `Future` returned to the asker.
This overhead of creating this intermediate actor is a good reason to avoid asking when simply telling can be sufficient.

# Conclusion

Actors and futures can be very powerful when combined in a single application.
The resulting application will be able to do most if not all operations concurrently without blocking.
There are some things to keep in mind when processing a `future` inside an actor, 
the recommended way of handling them is to always `pipe` them around.
The `ask` pattern is a nice tool to have when you need to link messages to responses but are often not necessary and should be avoided in those cases.

# Links

[^1]: https://docs.scala-lang.org/overviews/core/futures.html#execution-context
[^2]: https://doc.akka.io/docs/akka/current/actors.html#ask-send-and-receive-future
