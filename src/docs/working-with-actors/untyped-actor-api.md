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

## Receive messages
When an actor receives a message it is passed into the `OnReceive` method, this is an abstract method on the `UntypedActor` base class that needs to be defined.

Here is an example:
```csharp
public class MyUntypedActor : UntypedActor
{
    private ILoggingAdapter log = Context.GetLogger();

    protected override void OnReceive(object message)
    {
        if (message is string)
        {
            log.Info("Received String message: {0}", message);
            Sender.Tell(message, Self);
        }
        else
        {
            Unhandled(message);
        }
    }
}
```

## Become/Unbecome
### Upgrade
Akka supports hotswapping the Actor’s message loop (e.g. its implementation) at runtime. Use the `Context.Become` method from within the Actor. The hotswapped code is kept in a `Stack` which can be pushed (replacing or adding at the top) and popped.

> **Warning**<br>
Please note that the actor will revert to its original behavior when restarted by its Supervisor.

To hotswap the Actor using `Context.Become`:

```csharp
public class HotSwapActor : UntypedActor
{
    protected override void OnReceive(object message)
    {
        if (message.Equals("angry"))
        {
            Become(Angry);
        }
        else if (message.Equals("happy"))
        {
            Become(Happy);
        }
        else
        {
            Unhandled(message);
        }
    }

    private void Angry(object message)
    {
        if (message.Equals("angry"))
        {
            Sender.Tell("I am already angry!", Self);
        }
        else if (message.Equals("happy"))
        {
            Become(Happy);
        }
    }

    private void Happy(object message)
    {
        if (message.Equals("happy"))
        {
            Sender.Tell("I am already happy :-)", Self);
        }
        else if (message.Equals("angry"))
        {
            Become(Angry);
        }
    }
}
```

This variant of the `Become` method is useful for many different things, such as to implement a Finite State Machine (FSM). It will replace the current behavior (i.e. the top of the behavior stack), which means that you do not use Unbecome, instead always the next behavior is explicitly installed.

The other way of using `Become` does not replace but add to the top of the behavior stack. In this case care must be taken to ensure that the number of “pop” operations (i.e. `Unbecome`) matches the number of “push” ones in the long run, otherwise this amounts to a memory leak (which is why this behavior is not the default).

```csharp
public class Swapper : UntypedActor
{
    public static readonly object SWAP = new object();

    private ILoggingAdapter log = Context.GetLogger();

    protected override void OnReceive(object message)
    {
        if (message == SWAP)
        {
            log.Info("Hi");

            Become(m =>
            {
                log.Info("Ho");
                Unbecome();
            }, false);
        }
        else
        {
            Unhandled(message);
        }
    }
}
...

static void Main(string[] args)
{
    var system = ActorSystem.Create("MySystem");
    var swapper = system.ActorOf<Swapper>();

    swapper.Tell(Swapper.SWAP);
    swapper.Tell(Swapper.SWAP);
    swapper.Tell(Swapper.SWAP);
    swapper.Tell(Swapper.SWAP);
    swapper.Tell(Swapper.SWAP);
    swapper.Tell(Swapper.SWAP);

    Console.ReadLine();
}
```

## Stash

The `IActorStash` interface enables an actor to temporarily stash away messages that can not or should not be handled using the actor's current behavior. Upon changing the actor's message handler, i.e., right before invoking `Context.Become` or `Context.Unbecome`, all stashed messages can be "unstashed", thereby prepending them to the actor's mailbox. This way, the stashed messages can be processed in the same order as they have been received originally.

> **Note** <br>
The interface `IActorStash` extends the marker interface `IRequiresMessageQueue<IDequeBasedMessageQueueSemantics>` which requests the system to automatically choose a deque based mailbox implementation for the actor. If you want more control over the mailbox, see the documentation on mailboxes: Mailboxes.

Here is an example of the Stash in action:
```csharp
public class ActorWithProtocol : UntypedActor, IWithUnboundedStash
{
    protected override void OnReceive(object message)
    {
        if (message is string)
        {
            var str = (string)message;
            if (str.Equals("open"))
            {
                Stash.UnstashAll();
                Context.BecomeStacked(newMsg =>
                {
                    var newMsgStr = (string)newMsg;
                    if (newMsgStr.Equals("write"))
                    {
                        // do writing...
                    }
                    else if (newMsgStr.Equals("close"))
                    {
                        Stash.UnstashAll();
                        Context.UnbecomeStacked();
                    }
                    else
                    {
                        Stash.Stash();
                    }
                });
            }
            else
            {
                Stash.Stash();
            }
        }
    }

    public IStash Stash { get; set; }
}
```;
Invoking `Stash.Stash()` adds the current message (the message that the actor received last) to the actor's stash. It is typically invoked when handling the default case in the actor's message handler to stash messages that aren't handled by the other cases. It is illegal to stash the same message twice; to do so results in an `IllegalStateException` being thrown.

Invoking `Stash.UnstashAll()` enqueues messages from the stash to the actor's mailbox until the capacity of the mailbox (if any) has been reached (note that messages from the stash are prepended to the mailbox). In case a bounded mailbox overflows, a MessageQueueAppendFailedException is thrown. The stash is guaranteed to be empty after calling `Stash.UnstashAll()`.
