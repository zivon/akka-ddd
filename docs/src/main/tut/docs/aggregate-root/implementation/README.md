---
layout: docs
title: Aggregate Root - Implementation
permalink: /docs/aggregate-root/implementation
---

## Aggregate Root - Implementation

The Akka-DDD framework allows to model a behavior of an Aggregate Root as a state machine using the [Algebraic Data Type](). Each type must extend from the [Behavior]() trait and implement the `actions`. Please see the example below.

[Dummy]() is the base trait of the Dummy behavior ADT.

```scala
sealed trait Dummy extends Behavior[DummyEvent, Dummy, DummyConfig]
```

One of the possible states the Dummy can reach is the Active state. The behavior of Dummy in the Active state is implemented in the following way:

```scala
case class Active(value: Int, version: Long) extends Dummy {
    def actions =
      handleCommand {
          case ChangeValue(id, newValue) =>
            rejectNegative(newValue) orElse ValueChanged(id, newValue)
        
          case Reset(id) =>
            ValueChanged(id, 0)
      }
      .handleEvent {
          case ValueChanged(_, newValue) =>
            copy(value = newValue)
      }
}
```

The initial behavior of the Aggregate Root (with no associated state) should satisfy the [Uninitialized] type class, so that it can be automatically created by the framework. See example below:

```scala
implicit object Uninitialized extends Dummy with Uninitialized[Dummy] {

    def actions: Actions =
      handleCommand {
        case CreateDummy(id, name, description, value) =>
          rejectNegative(value) orElse
            DummyCreated(id, name, description, value)
      }.handleEvent {
          case DummyCreated(_, _, _, value) =>
            Active(value, 0)
      }
}
```

The behavior (the command processing logic) consists of two parts: **command handling (reaction)** and **event handling (state transition)**. The `actions` method is a factory of the Command Handler and Event Handler functions. It creates an instance of [Actions]() that contains Command Handler and Event Handler functions.

## Command Handling (reaction)

The actual command handling logic (the Command Handler) is a partial function that accepts a `Command` and returns a `Reaction`. The `Reaction`, that indicates the acceptance of a command, can be constructed from an event or a sequence of events. The `Reaction`, that indicates the [rejection]() of a command, can be created by providing a rejection reason. 

### Command Acceptance

The Command Handler indicates that the command is accepted by returning an Event.

```scala
case AddParticipant(id, name) => ParticipantAdded(name, id)
```

Sometimes a single event is not enough. You can build a sequence of events in many ways, for example by transforming an existing sequence of an arbitrary type.

```scala
case RemoveAllParticipants(id) =>
  this.participants  
    .map { name => ParticipantRemoved(name, id) }

// Notice that `participants` is a property of the current state.
```

You can also use a special '&' operator to build a sequence of events from scratch.

```scala
case Reset(id, name) =>
  NameChanged(id, name) & ValueChanged(id, 0, version + 1)

// Notice that `version` is a property of the current state.
```

### Command Rejection 

There are two ways to reject a command. The first way is to omit the command handler implementation. In this case, the command will be rejected automatically with a rejection reason of type [CommandHandlerNotDefined]().

To indicate explicitly that a command must be rejected (for example due to a validation error), the command handler should return a rejection reason of type [DomainException]() or its subtype. 

The `reject` method should be used to create a rejection: 

```scala
case anyCommand  =>
  reject (LotteryHasAlreadyAWinner(s"Lottery has already a winner and the winner is $winner"))

```

It is also possible to pass a message to the `reject` method. In this case a rejection of type [DomainException]() will be created automatically:

```scala
case Run(_)  =>
  reject("Lottery has no participants")

```

To keep the implementation of the command handler concise, `rejectIf` method can be used as shown below: 

```scala
case CreateDummy(id, name, description, value) =>
  rejectIf(value < 0, "negative value not allowed") orElse
    DummyCreated(id, name, description, value)

```

Again you can either pass a message or a custom rejection reason to the `rejectIf` method. 


### Collaboration with other actors

Sometimes the command handling logic is more complex and requires the AR Actor to collaborate with other Actors. In such cases the `Reaction` needs to be implemented as a method of the AR Actor and exposed as a property of type `Reaction` of the [Aggregate Root configuration]() object. The command handler will simply return the `Reaction` that is available in the AR configuration as shown in the example below ([Dummy]() behavior): 

```scala
case GenerateValue(_) =>
  ctx.config.valueGeneration
```
See also: [Aggregate Root Actor - Collaboration with other actors]()

## Event handling (state transition)

The Event Handler is a partial function that accepts Event and returns Behavior.

```scala
case ParticipantRemoved(name, id) =>
  val newParticipants = participants.filter(_ != name)
  // NOTE: if last participant is removed, transition back to EmptyLottery
  if (newParticipants.isEmpty)
    EmptyLottery
  else
    copy(participants = newParticipants)
```

In the example above, the result is either a new version of the current behavior (with modified `participants` property) or completely new behavior (`EmptyLottery`).

## Complete examples

* [Dummy](https://github.com/pawelkaczor/akka-ddd/blob/master/akka-ddd-test/src/test/scala/pl/newicom/dddd/test/dummy/DummyAggregateRoot.scala)
* [Reservation](https://github.com/pawelkaczor/ddd-leaven-akka-v2/blob/master/sales/write-back/src/main/scala/ecommerce/sales/ReservationAggregateRoot.scala)
* [Lottery](https://github.com/pawelkaczor/akka-ddd/blob/master/akka-ddd-test/src/test/scala/lottery/domain/model/Lottery.scala) 
* [Document](https://github.com/pawelkaczor/akka-ddd/blob/master/akka-ddd-test/src/test/scala/pl/newicom/dddd/test/dms/DocumentAR.scala)

## Behavior composition

TODO