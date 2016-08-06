---
layout: docs.hbs
title: Actor API
---
# UntypedActor API
The `UntypedActor` class defines only one abstract method, the above mentioned `OnReceive(object message)`, which implements the behavior of the actor.

If the current actor behavior does not match a received message, it's recommended that you call the unhandled method, which by default publishes a new `Akka.Actor.UnhandledMessage(message, sender, recipient)` on the actor system’s event stream (set configuration item `Unhandled` to on to have them converted into actual `Debug` messages).

In addition, it offers:

* `Self` reference to the `IActorRef` of the actor

* `Sender` reference sender Actor of the last received message, typically used as described in [Reply to messages](reply-to-messages).

* `SupervisorStrategy` user overridable definition the strategy to use for supervising child actors

This strategy is typically declared inside the actor in order to have access to the actor’s internal state within the decider function: since failure is communicated as a message sent to the supervisor and processed like other messages (albeit outside of the normal behavior), all values and variables within the actor are available, as is the `Sender` reference (which will be the immediate child reporting the failure; if the original failure occurred within a distant descendant it is still reported one level up at a time).

* `Context` exposes contextual information for the actor and the current message, such as:

  * factory methods to create child actors (`ActorOf`)
  * system that the actor belongs to
  * parent supervisor
  * supervised children
  * lifecycle monitoring
  * hotswap behavior stack

The remaining visible methods are user-overridable life-cycle hooks which are described in the following:

```csharp
public override void PreStart()
{
}

protected override void PreRestart(Exception reason, object message)
{
    foreach (IActorRef each in Context.GetChildren())
    {
      Context.Unwatch(each);
      Context.Stop(each);
    }
    PostStop();
}

protected override void PostRestart(Exception reason)
{
    PreStart();
}

protected override void PostStop()
{
}
```
The implementations shown above are the defaults provided by the `UntypedActor` class.
