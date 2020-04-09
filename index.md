---
layout: post
title: 'Event Sourcing: Awesome, powerful & different'
date: '2016-10-24 16:55:00'
tags:
- event-sourcing
cover: '/images/2019/01/2016-10-24-Achtergrond-3.1.svg.png'
---

I was introduced to Event Sourcing by the [<span class="smallcaps">CQRS</span> Journey](https://msdn.microsoft.com/en-us/library/jj591559.aspx?f=255&MSPPError=-2147217396#) series of articles from Microsoft’s Patterns and Practices team. It’s an excellent read about a pattern that offers great flexibility, at a cost. Event Sourcing was mentioned as more of a side note, but I was immediately drawn to its potential. As I’m currently involved in developing an application which uses <span class="smallcaps"><abbr title="Command-Query Responsibility Segregation">CQRS</abbr></span> and Event Sourcing, I’ll be journaling my understanding of it and its benefits and disadvantages.

## What is Event Sourcing?

There are several excellent articles about this subject already, so I might be repeating some information. This is _my_ understanding of the paradigm.

The ‘traditional’ way of persisting the state of your application is to  store the state as it currently is. For example, your application might be a calendar, so you want to store appointments. It might look something like this:

<div>This is a DIV</div>

<div class="nobreaktable"><table><thead><tr><th>Appointment ID</th><th>Start time</th><th>End time</th><th>Title</th></thead><tbody><tr><td style="text-align:right">1</td><td>09:30</td><td>10:45</td><td>Sprint Review</td></tr><tr><td style="text-align:right">2</td><td>11:15</td><td>11:30</td><td>Discuss Sprint Review with Stakeholders</td></tr></tbody></table></div>

This is a table that should look pretty familiar. It might be a table in a relational database, or a set of documents in a key-value store, or even a list of objects held in memory. The point is, it represents the state of system as it is now. Let me ask you a question: how did the system get to this state?

### The audit log

To answer that question, you could create an audit log. In addition to creating or updating an appointment, you also store a record in the audit log, describing what happened (an ‘event’). For a single appointment, it might look something like this:

<div class="nobreaktable">
<table><thead><tr><th>Sequence</th><th>Event</th></thead>
<tbody>
<tr><td style="text-align:right">1</td><td>Appointment created:
<ul style="margin-bottom:0"><li>09:30 - 10:30</li><li>Title: Review Sprint</li></ul></td></tr>
<tr><td style="text-align:right">2</td><td>Appointment rescheduled: 09:30 - 10:45</td></tr>
<tr><td style="text-align:right">3</td><td>Title changed: Sprint Review</td></tr>
</tbody></table>
</div>

This reveals a whole lot of information that simply isn’t there if you only have the current state of the system. After it was created, the appointment was rescheduled once and its title was changed. The initial end time of 10:30 was there after the appointment was just created, but after it was rescheduled, that information _disappeared_. Poof.

This kind of information can be pretty useful for users of your application, because they can see what happened and who they should blame. Developers will profit the most from it; it’s so much easier to analyze a problem when you can see _why_ something is in a particular state.

### Single source of truth
An audit log beside your current state is useful, but there’s a problem: conflict. If you find yourself in a situation where your current state says A and your audit log says B, you now have two problems. You not only have to diagnose the problem at hand, but you also have to figure out where the discrepancy comes from.

The problem is the fact that you have two sources of ‘truth’ telling you what the state of the system _should_ be. Your application only looks at the current state, so it sees no problem. You as a developer have two sources of truth. Because they conflict, neither can really be trusted.

### Enter Event Sourcing
If we eliminate the audit log, we have one source of truth left, but we lose all of the detailed historical information that we value so much. What if we eliminate the current state?

That is the essence of Event Sourcing: instead of storing the current state of a system, you only store the events that lead up to that state. To get the current state, you can ‘replay’ those events in memory.

This current state is only an ‘ephemeral’ representation of events that have happened. It is not persisted. Well, there are snapshots&mdash;more on that later&mdash;but they are discardable. You can change what events do to the state of your application, but you cannot change the events. They have become the irrefutable truth.

### Replaying?

To get the current state of your system, you will have to replay all the events. All the events? Well, not _really_ all of them. Your current state is usually divided into several logical ‘objects’, which would traditionally be rows in a table. If you uniquely identify each object, you only need to replay the events for that object to get that object’s state.

Let’s look at a very simplistic example of the current state for our calendar. 

```csharp
class Appointment
{
    public Guid AppointmentId { get; }
    public DateTime StartTime { get; private set; }
    public DateTime EndTime { get; private set; }
    public string Title { get; private set; }
    public bool IsCanceled { get; private set; }
}
```

The events look like this.

```csharp
class AppointmentCreated
{
    public Guid AppointmentId { get; }
    public DateTimeOffset StartTime { get; }
    public DateTimeOffset EndTime { get; }
    public string Title { get; }
}

class AppointmentRescheduled
{
    public DateTimeOffset StartTime { get; }
    public DateTimeOffset EndTime { get; }
}

class AppointmentRenamed
{
    public string Title { get; }
}

class AppointmentCanceled
{
}
```

Note that the `Appointment​Canceled` event does not have any properties. Just its type is a meaningful representation of what has happened. 

What would replaying those events look like from the point of view of `Appointment`? Like this.

```csharp
void ReplayEvent(AppointmentCreated @event)
{
    AppointmentId = @event.AppointmentId;
    StartTime = @event.StartTime;
    EndTime = @event.EndTime;
    Title = @event.Title;
}

void ReplayEvent(AppointmentRescheduled @event)
{
    StartTime = @event.StartTime;
    EndTime = @event.EndTime;
}

void ReplayEvent(AppointmentRenamed @event)
{
    Title = @event.Title;
}

void ReplayEvent(AppointmentCanceled @event)
{
    IsCanceled = true;
}
```
You start with a blank object. Then you simply call `Replay​Event` for each event that has happened, in the order they have happened. That’s all you need to recreate the current state.

## Modifying state
Since state is represented by the events that have happened in the past, modifying that state is done by appending events. Making this the ‘current state’ object’s responsibility is a convenient way of keeping all the knowledge about the business logic and its rules in one place, which is a philosophy you also see in <span class="smallcaps"><abbr title="Domain Driven Design">DDD</abbr></span>. 

The current state object (or entity object or domain object, however you want to call it) also becomes responsible for its own consistency; it is the only actor creating events about itself and it’s able to use events from the past to allow or disallow operations: your appointment has _already_ been canceled; you cannot cancel it again.

This could look something like this.

```csharp
class Appointment
{
    public Appointment(
        Guid id,
        DateTimeOffset startTime,
        DateTimeOffset endTime,
        string title)
    {
        if(endTime < startTime)
            throw new EndTimeBeforeStartTimeException();

        AppendEvent(
            new AppointmentCreated(id, startTime, endTime, title)
        );
    }

    public void Reschedule(
        DateTimeOffset startTime,
        DateTimeOffset endTime)
    {
        AppendEvent(
            new AppointmentRescheduled(startTime, endTime)
        );
    }

    public void Cancel()
    {
        if (IsCanceled)
            throw new AppointmentAlreadyCanceledException();

        AppendEvent(
            new AppointmentCanceled()
        );
    }
}
```

In this example, `Append​Event` is a method which will append an event to storage for the unique object that the `Appointment` refers to.

Validation of the parameters is done inside the methods. It makes a method ultimately responsible for the validity and consistency of the data. For brevity, `Reschedule` does not validate the parameters, but `Cancel` does check if the operation is allowed given the current state.

Currently, if you attempt to cancel an appointment that has already been canceled, an exception will be thrown. This doesn’t need to be the case. We could skip the check and just store the event anyway. It does not change the fact it has been canceled; we will still set the `Is​Canceled` property to `true` when it is replayed. The point here is: you do not always _need_ to maintain and validate against state. Sometimes it is sufficient to ignore superfluous events.

Notice that all that the constructor and the `Reschedule` and `Cancel` methods are actually _doing_ is storing an event. They are not directly modifying the state. Why? Well, we already have a method to modify the state based on an event that has happened: the `Replay​Event` method. So, in addition to storing an event, `Append​Event` will also immediately replay the event, so its state is up to date for any other operations you might want to perform and we don’t have to write the code to change state twice.

## What does it bring to the table?
Let’s establish some ground rules about events.

- Events describe what *has already happened*, and because we cannot change the past, they are **immutable**. Events cannot be changed after they are persisted.
- Extending on the previous rule: they cannot be deleted.
- Each event contains all the information needed to represent the state change and to be able to replay it. They should not include computed values if they can be derived from the event, possibly when combined with earlier events.
- An event’s metadata contains its type and when it occurred, among others.
- Events can be uniquely identified by the identity of the object they apply to and an ordinal&mdash;also referred to as the revision or version number.
- You do not query over events. They are only used to replay into a representation of the state of the system at a given time.

For an event-sourced object, a request to change state should only result in one of three things:

- Storing an event;
- Throwing an exception (because a business rule would be violated);
- Doing nothing.

Other results, such as directly modifying the object’s state or making a database call, are strongly discouraged because they are side effects. Side effects are usually not represented in the events that are stored, which means you cannot replay them. They also make testing a lot harder.

Considering these rules, let’s look at some obvious, and not so obvious, benefits and disadvantages.

### Benefits
Because you only have to read and append events, storage is very easy. The only things you need to be able to do are appending an event and retrieving the list of events for a unique object. You can store events pretty much anywhere: in a table, a key-value store, a directory of files, an e-mail, or almost anywhere you can imagine. Since events are immutable, they are very easy to cache, which enables you to get excellent performance.

You will not need _explicit_ transactions any more, because you are only inserting data, not updating or deleting it; you also only need a single table, collection, set, or however your data store groups data. A transaction is like a lock, so not requiring transactions is kind of like lock-free code: there are no deadlocks and it’s _much_ faster. This assumes that you design your events and group them into unique objects in such a way that you never need to manipulate more than one of them at a time&mdash;which is not that difficult to achieve.

You also get an audit log for free, which you can use when debugging to see exactly what has happened and why the system is in a particular state.

If you make sure the operations on your entity class are free from side effects, it basically lives in isolation and the class will have a very low [coupling metric](https://en.wikipedia.org/wiki/Efferent_coupling). You’ve isolated that part of the business logic from the rest of the system.

If you’ve achieved this, testing your logic can be expressed as ‘_given_ these events have happened, _when_ this operation is requested, _then_ do these events get stored?’ Or, ‘_then_ does this exception get thrown?’ Given, when, then. Does that sound familiar?

You don’t have to deal with the fact that relational databases are built on set algebra, while your objects are not. Event Sourcing sidesteps the object-relational impedance mismatch.

Your source of truth is a bunch of events in a database. After you store them, you can also _broadcast_ them to the rest of the world via a messaging system. It’s very easy to let other applications know what’s happening in your system. You no longer need to query into another system’s data store and duplicate it to find out what has changed. All you need to store is a list of unique object identifiers and their latest known version number.
### Disadvantages
Of course there are downsides to every approach, and Event Sourcing is no exception. The most important practical disadvantage, one which you will keep encountering when using Event Sourcing, is this: there is no easy way to see what the persisted state of your application is, and you cannot query over it. The only time when you can see what happens is at run-time. A solution, which is outside of the scope of this post, is using a separate model for reading that responds to the events that happen in your system. The <span class="smallcaps"><abbr title="Command-Query Responsibility Segregation">CQRS</abbr></span> pattern combines very well with Event Sourcing to accomplish this.

There are also disadvantages originating from the fact that Event Sourcing is an entirely different approach to what most people are used to. It will take some getting used to. It also takes a bit more boiler-plating than ‘regular’ current-state backing. You need to define each event that can take place in your system, so adding new features might be slower than you’re used to. To offset that, your data becomes tons more valuable.

Event Sourcing sounds like a very inefficient mechanism to store state. It is; it requires a lot more storage than current-state backing. It also requires more processing time, simply because you need to retrieve more data to get to the current state. Because you have practically limitless amounts of storage and processing power nowadays, it’s pretty easy to overcome any problems that arise from these ineffiencies. One of those is the use of snapshots.

## Snapshots
You might think that when an event stream contains thousands or tens of thousands of events, your system must become slow, because for each operation it needs to replay all of those events. Well, not necessarily.

Remember, the only thing replaying events is doing is modifying some representation of state. It should be idempotent; whether you replay the events once or a hundred times, it should always have the exact same outcome.

After we’ve replayed, let’s say, ten thousand events, we can take a snapshot of the resulting state and store that, along with the identity of the last replayed event. Now, when we want the current state, we simply load the snapshot and replay any event that has happened since the snapshot, of which there should only be a few. If you’ve created your snapshot correctly, the outcome of loading a snapshot and replaying new events should be _identical_ to replaying all of the events.

It doesn't invalidate the events that happened. If, for some reason, the representation of state has changed, we can simply discard the snapshot and create a new one by replaying all the events at that point.

## Testing
I mentioned that testing becomes a lot easier. Let’s make this a little more concrete and show _how_ you test an event-sourced object.

```csharp
private AppointmentCreated CreateAppointmentCreatedEvent()
{
    return new AppointmentCreated(
        appointmentId: Guid.NewGuid(),
        startTime: DateTimeOffset.Now.AddHours(3),
        endTime: DateTimeOffset.Now.AddHours(4),
        title: "Appointment"
    );
}

private AppointmentRenamed CreateAppointmentRenamedEvent()
{
    return new AppointmentRenamed(title: "Renamed appointment");
}

private Appointment CreateSut(IEnumerable<IEvent> events)
{
    var sut = new Appointment();
    
    foreach (var @event in events)
        sut.ReplayEvent(@event);
    
    return sut;
}

[Fact]
public void Reschedule_AppendsAppointmentRescheduled()
{
    // GIVEN these events have happened
    var events = new IEvent[]
                 {
                     CreateAppointmentCreatedEvent(),
                     CreateAppointmentRenamedEvent()
                 };
    var sut = CreateSut(events);
    
    // WHEN we ask to reschedule
    var newStartTime = DateTimeOffset.Now.AddHours(5);
    var newEndTime = DateTimeOffset.Now.AddHours(6);
    sut.Reschedule(newStartTime, newEndTime);
    
    // THEN does the AppointmentRescheduledEvent get published?
    sut.AppendedEvents.ShouldAllBeEquivalentTo(
        new[]
        {
            new AppointmentRescheduled(newStartTime, newEndTime)
        },
        config => config.RespectingRuntimeTypes()
    );
}
```
This exposes a few details I haven’t touched upon. They are mostly implementation details; they’re _my_ way of solving some problems, you might choose a different way.

Remember `AppendEvent`?
>`Append​Event` is a method which will append an event to storage for the unique object that the `Appointment` refers to.

That’s a little white lie. From the perspective of the `Appointment`, that is what happens, but it’s not _really_ what happens. 

For testability and decoupling, what really happens is that the event gets appended to a list of ‘unsaved’ events called `Appended​Events`. This enables us to very easily inspect what an object does and it enables the test you see above.

Note that the test can be divided into three logical sections: _given_ some state, _when_ an action is performed, _then_ can we observe this behavior? A language which makes this much more formal is **Cucumber** (look into it, it’s awesome for behavior testing). Because this post is already getting long in the tooth, you’ll find an example of how to do this in [this Gist](https://gist.github.com/heemskerkerik/72ee0895c3f88c59a60071c3b61ef072).

## Analogies
Event Sourcing is very powerful way of thinking, but it is not really new. There are event-sourcing systems most of us are familiar with and maybe don't recognize them as such. Some examples:

- Your bank account is _probably_ stored using Event Sourcing. The current account balance is simply a projection of all the deposits and withdrawals that you made.
- Version control systems. Each commit or change to a file is an event. If you replay all of the events, you get the current state of the source code.
- Most large <span class="smallcaps"><abbr title="Relational Database Management System">RDBMS</abbr></span>es use Event Sourcing under the hood. There are, simply put, only three events: insert, update and delete. The <span class="smallcaps">RDBMS</span> stores the events in the transaction log and then applies them to the tables.

## Conclusion
This is my understanding of Event Sourcing. I think it’s great, it’s powerful, but I might be biased because I’m already actively using it in an application under development. I hope you can see its potential. If so, please check out these excellent posts about the subject:

- [Rinat Abdullin: Why Event Sourcing?](https://abdullin.com/post/event-sourcing-why/)
- [Jonathan Oliver: Aggregate Roots and Shared Data](http://blog.jonathanoliver.com/aggregate-roots-and-shared-data/)
- [Rinat Abdullin: Event Sourcing - Versioning](https://abdullin.com/post/event-sourcing-versioning/)
- [Greg Young: <span class="smallcaps">CQRS</span> and Event Sourcing](http://codebetter.com/gregyoung/2010/02/13/cqrs-and-event-sourcing/)
- [Martin Fowler: Event Sourcing](http://martinfowler.com/eaaDev/EventSourcing.html)

In future posts I’ll be writing about my experience creating an application using Event Sourcing and <span class="smallcaps">CQRS</span>. Have any other advantages or disadvantages? Let me know!