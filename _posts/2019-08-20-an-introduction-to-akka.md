---
layout: post
title: "A Gentle Introduction to Akka"
description: "A brief introduction to Akka Actors."
comments: true
keywords: "akka, actors, actor model, introduction, intro"
---

This post will introduce the essential concepts in Akka actors I had to learn when I got started with it.
Code examples are written in Scala but the concepts behind them should be applicable to other languages as well.

# What is Akka?

Akka is a framework which allows you to build applications using the actor model.
The actor model makes it easy to design and implement highly concurrent and scalable applications.
A central concept in the actor model is *message passing* which replaces method calls between object.
Objects which send and receive messages are called *actors*. 

# What is an actor?

In terms of implementation, an actor is an instance of a class which extends the `Actor` trait provided by Akka and implements the `receive` method.

```scala
class Myactor extends Actor {
  def receive: Receive = ???
}
```

Up to some degree you can think about actors like regular classes, except for the fact that their methods are not called directly but interaction with them is done by sending them a message.
A message can be any object, as long as it is immutable. 
Where for a method call you would provide arguments to pass along data, you can send an object containing data as a message to an actor.
Say we want our actor to multiply two integers, we can create a case class specific for this action which requires the required data.

```scala
class MyActor extends Actor {
  import MyActor._

  def receive: Receive = ???
}

object MyActor {
  case class Multiply(left: Int, right: Int)
}
```

How do we then make our actor react to these messages? Tis is where the `receive` method comes into play. 
The receive method is of the `Receive` type, which is equal to `PartialFunction[Any, Unit]`.
Hence, we can define behaviour for any possible object and don't have to return anything.
In the actor model we typically send back a message instead of returning a value, but we could also do something like sending a message to another actor, an IO operation or something else.

For our example, `MyActor` can respond to a `Multiply` message as follows.

```scala
class MyActor extends Actor {
  import MyActor._

  def receive: Receive = {
    case Multiply(l, r) => println(s"$l * $r = ${l * r}")
  }
}

object MyActor {
  case class Multiply(left: Int, right: Int)
}
```

We can send a message to an actor as follows.

```scala
myActor ! Multiply(3, 5) // Our actor will print 3 * 5 = 15
```

It is generally a good idea to wrap the data you want to send to your actor in some case class.
Here, the `!` operator does **not** represent a direct call to `myActor.receive`.
In fact, `myActor` is in this example not an object of class `MyActor`, but an `ActorRef`.
You can think of an `ActorRef` as the address to send the message to, much like a postal address for a letter.
To understand how an actor responds to receiving messages, we need to talk about the *mailbox* concept.

Each actor has its own mailbox where all incoming messages are being stored.
These messages are being stored in the same order as they were sent, regardless of who the sender of the message is.
The receiving actor will always take the oldest message in its mailbox, process it and when it is done take the next message from its mailbox.
Hence, messages are processed first in, first out and one-by-one.

## The interior of an actor

A notable property of actors is that their interiors are not exposed to the outside world. 
Any state inside an actor cannot be accessed directly, the only thing we can observe from an actor are the messages they send back.
Actors thus can be, and often are, stateful. Their internal state may even be mutable but it is not shared.
This property is an important reason why actor systems are so well-suited for concurrent applications.
There are no long-lasting interactions between objects, only fire-and-forget message sending and no shared mutable state between actors,
which makes that actors can safely do computations concurrently.

Within the scope of an actor, an important value called `context` is available which provides contextual information on the actor such as:
* The parent of the actor
* The system the actor belongs to
* An `ActorRef` to the sender of the current message that is being processed
* The `ActorRef` of the actor itself
* A method to create other actors
* A method to change the behaviour of this actor

## Changing the behaviour of an actor

A common phenomenon in actors is that we might want to change the behaviour of actors in an event-driven way.
For instance, in our example above we might only want our actor to perform multiplications after we've told it to do so.
This can be accomplished using `context.become`.
Actors allow their behaviour to be changed dynamically at runtime as shown below.

```scala
class MyActor extends Actor {
  import MyActor._

  def receive = {
    case StartMultiply => context.become(readyToMultiply)
  }

  def readyToMultiply = {
    case Multiply(l, r) => println(s"$l * $r = ${l * r}")
  }
}

object MyActor {
  case class Multiply(left: Int, right: Int)
  
  case object StartMultiply
}
```

Now whe can control when our actor will perform calculations!

```scala
// ignore how this creation is done for now, will be explained later

val myActor = system.actorOf(Props[MyActor]) 

myActor ! Multiply(3, 5) // Nothing will happen

myActor ! StartMultiply

myActor ! Multiply(4, 6) // Actor will print 4 * 6 = 24

```

In the example above, `readyToMultiply` takes no parameters, but it is possible and often a good idea to pass parameters to the new state.
These parameters will then be in scope untill the actor `become`'s something else.

```scala
class MyActor extends Actor {
  import MyActor._

  def receive = {
    case StartMultiply(factor: Int) => context.become(readyToMultiply(factor))
  }
  
  def readyToMultiply(factor: Int) = {
    case Multiply(i: Int) => println(s"$factor * $i = ${factor * i}")
    case StartMultiply(i: Int) => context.become(readyToMultiply(i))
  }
}

object MyActor {
  case class Multiply(i: Int)
  case class StartMultiply(i: Int)
}
```

```scala
val myActor = system.actorOf(Props[MyActor])

myActor ! StartMultiply(2)

myActor ! Multiply(4) // prints 2 * 4 = 8


myActor ! StartMultiply(3)

myActor ! Multiply(4) // prints 3 * 4 = 12
```


# The actor tree 

An important property of actor systems is that the actors are structured in a tree.
Each actor has a parent actor and has 0 or more child actors.
Actors can be created dynamically at runtime.
The creation of actors is a cheap operation and it is not an uncommon pattern to spawn an actor to execute a small task and to stop it after this task is finished.
When an actor is stopped, all its children will be stopped as well to prevent the existence of orphaned actors.
A new actor is created by calling the `.actorOf()` method and passing it a `Props` object.
The Akka docs describe `Props` as follows

> Props is a configuration class to specify options for the creation of actors, 
think of it as an immutable and thus freely shareable recipe for creating an actor including associated deployment information. [[^1]]

The `actorOf` method can be called on the root node of the actor system, usually called `system` or on the `context` value within an actor.
Calling it on `system` will give you a top-level actor in your application and is usually done in `Main`.
Calling it on the `context` within an actor will make the created actor a child of the actor which does the creation.
An actor can be created as follows.

```scala
class ChildActor extends Actor {
  ...
}

class MyActor extends Actor {
  val child = context.actorOf(Props[ChildActor]) 
  
  child ! "Hello" // child is an ActorRef to the child
  
  ...
}
```

It is recommended practice to create a method in a companion object of an actor which creates a `Props` object for an actor. [[^2]]

## Supervision and Fault Tolerance

A very important property of the parent-child relationships that exist between actors is that parents *supervise* their children.
This means that when an exception occurs in an actor, its parent will receive a notification of this.
This parent will see the type of exception that was thrown by its child and can decide what to do.
It has four options:
* `Resume` the child, keeping its accumulated internal state
* `Restart` the child, clearing out its accumulated internal state but keeping the mailbox of the actor intact
* `Stop` the child
* `Escalate` the exception by re-throwing it, thus propagating the decision to its own parent

The default choice is `Restart`. 
This has the implication that an uncaught exception will by default not bring down your application 
(but it can bring your actor in a permanent retry loop if you are not careful)!
The way an actor decides what to do with an exception thrown by a child is called its `SupervisionStartegy` and may include things like retry mechanisms.


[^1]: https://doc.akka.io/docs/akka/current/actors.html#props
[^2]: https://doc.akka.io/docs/akka/current/actors.html#recommended-practices