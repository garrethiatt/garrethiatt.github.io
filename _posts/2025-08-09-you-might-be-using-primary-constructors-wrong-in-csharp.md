---
layout: post
title: You might be using primary constructors wrong in C#
date: 2025-08-09 17:02 -0600
categories:
- Coding
- C#
tag:
- ".net"
- c#
- design patterns
- style
description: This post goes over the primary constructor feature in C#12 and why you may not want to use it
---

## TL;DR
Don't use primary constructors for `class` and `struct` types. Only use them for `record` and `record struct` types because that is
what they were originally intended and best designed for. When used with classes and structs, it just opens the door to many undesired
behaviors and bad practices. 

## A Brief History
Primary constructors have been around since C#10 when `record` was introduced. It made perfect sense that records and primary 
constructors came together because of the nature of how `record` works and how they are used. However, when C#12 introduced primary 
constructors to `struct` and `class` types, it (heavy air quotes) "simplified" constructors. For example, prior to C#12, you may 
have had a class like this, which ensured properties were set when the class was constructed by requiring parameters that assigned 
them within the constructor.

```csharp
public class Customer
{
    public string FirstName { get; set; }
    public string LastName { get; set; }

    public Customer(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
    }
}
```
{: .nolineno }

With primary constructors, this same class could be simplified down to something like this instead

```csharp
public class Customer(string firstName, string lastName)
{
    public string FirstName { get; set; } = firstName;
    public string LastName { get; set; } = lastName;
}
```
{: .nolineno }

And maybe for very simple classes, that works and makes sense. It looks clean and is easy to read, but once you start applying 
primary constructors in other use cases, things get out of hand very quickly.

## Microsoft's rules about primary constructors and misperceptions
There are a few [rules](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/primary-constructors#understand-rules-for-primary-constructors){:target="_blank"}
that Microsoft states about primary constructors which muddy the waters and are somewhat misleading

### Primary constructor parameters might not be stored if they aren't needed
Basically, if a primary constructor takes a parameter, but doesn't use it, it's not stored. What does this mean exactly?

```csharp
public class Customer(string firstName, string lastName)
{
    // only firstName is used, so lastName is never part of the compiled object
    public string FullName { get; set; } = firstName; 
}
```
{: .nolineno }

```csharp
public class Customer(string firstName, string lastName)
{
    // lastName is used, so the compiler will create both parameters
    public string FullName { get; set; } = firstName + " " + lastName; 
}
```
{: .nolineno }

This is really all just about how the compiler handles this class. But it doesn't really add much value to the developer. 
You'll be warned about the first example that `lastName` is unread. Why would you ask for this parameter in the constructor only 
to never use it?

### Primary constructor parameters aren't members of the class
This is one of the more confusing statements. Yes, parameters in the constructor don't become "members" (heavy air quotes again), 
but all that means is that the *instance* of your class can't directly access it like a member via `this` keyword. That's actually 
good because if you never explicitly declared a member and could magically access it within the instance of my class, that would be 
unexpected. However, this rule from Microsoft is misleading because although primary constructor parameters don't become members
and they are not properties, all of the used parameters in a primary constructor are *in scope* of the class and can be directly 
accessed and *modified* (more on that later) within the class. They're basically compiled into private hidden fields within the class. 

### Primary constructor parameters can be assigned to
If there were no other reasons to *not* use primary constructors in classes, this one is convincing enough on its own. There's 
something especially concerning about allowing a parameter to go into scope of a class, and be accessible *throughout* the class, 
and be mutable.

Here's a simple example from [Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/primary-constructors#use-dependency-injection){:target="_blank"}
that shows how to use primary constructors for dependency injection.

```csharp
public interface IService
{
    Distance GetDistance();
}

public class ExampleController(IService service) : ControllerBase
{
    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return service.GetDistance();
    }
}

// The Microsoft example doesn't provide this, but let's assume this is the Distance class
public class Distance
{
    public int Miles { get; set; }
}

// Let's assume this is the existing implementation of the IService interface that's registered 
// at startup and injected via .NET DI
public class DistanceService : IService
{
    public Distance GetDistance()
    {
        return new() { Miles = 10 };
    }
}
```
{: .nolineno }

This is seemingly straightforward and generally makes sense. However, let's consider if another method is added to the controller and
someone decides to write a new implementation of `IService` and do this.

```csharp
// New implementation of IService (not registered at startup to inject in .NET DI)
public class NewDistanceService : IService
{
    public Distance GetDistance()
    {
        return new() { Miles = 1000 };
    }
}

public class ExampleController(IService service) : ControllerBase
{
    // New method to get distance from another service
    [HttpGet]
    public ActionResult<Distance> GetOtherDistance()
    {
        service = new OtherDistanceService(); // service is reassigned with newly written service
        return service.GetDistance();
    }

    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return service.GetDistance();
    }
}
```
{: .nolineno }

This would work and is completely allowed with primary constructors. But consider what this is doing. Although `IDistance` is
already configured within .NET DI to inject the `DistanceService` implementation to `service`, `GetOtherDistance()` *modifies* it 
to another implementation through `OtherDistanceService`. Consider if these methods are called in this order:
- `Get()` is called and returns Distance with 10 miles
- `GetOtherDistance()` is called and returns a Distance with 1000 miles
- `Get()` is called and returns a Distance with ***1000 miles***

This is because `service` was modified to a different implementation between calls and now `Get()` will return an unexpected
value.

> Controllers may not be the best example of this because service lifetimes of DI services. A new instance of the controller would be
> created per request, meaning `service` wouldn't change between method calls but the point remains that this pattern could very much
> negatively impact many other services in the request pipeline.
{: .prompt-warning}

The fact that parameters in primary constructors are not immutable is one of the biggests reasons I advocate *against*
primary constructors.

Now, you may argue a couple of points on this. First, even before primary constructors, devs could assign an injected dependency to
a private field that is *not* readonly and encounter the same problem. I agree, but this anti-pattern is far more familiar and easy 
to spot when comparing to primary constructors. Secondly, you could argue that this could be mitigated by making `service` immutable 
by adding `in` keyword in the primary constructor parameter. This would then *require* it to be assigned to a private field to 
be available in the class because it's not available to members.

```csharp
public class ExampleController(in IService service) : ControllerBase // Add 'in' keyword to service
{
    [HttpGet]
    public ActionResult<Distance> GetOtherDistance()
    {
        service = new OtherDistanceService(); // This would not be allowed 
        return service.GetDistance(); // service would be innaccessible here
    }

    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return service.GetDistance(); // service would be inaccessible here
    }
}
```
{: .nolineno }

You could then assign it to a `readonly` field. Both `_service` and `service` would be immutable.

```csharp
// Add 'in' keyword to service to make it readonly
public class ExampleController(in IService service) : ControllerBase 
{
    private readonly IService _service = service; // Assign within initialization

    [HttpGet]
    public ActionResult<Distance> GetOtherDistance()
    {
        // This would not be allowed because _service is readonly
        _service = new OtherDistanceService(); 
        return _service.GetDistance();
    }

    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return _service.GetDistance(); 
    }
}
```
{: .nolineno }

> Although it would work, there are drawbacks of using the 
> [`in`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters#in-parameter-modifier){:target="_blank"}
> keyword this way, so I would *not* recommend this.
{: .prompt-warning}

We want to generally avoid `in` keyword on primary constructor parameters but also want to enforce immutability on a dependency,
so what then? Should we follow a pattern similar to the traditional constructor and assign to a readonly field?

```csharp
public class ExampleController(IService service) : ControllerBase
{
    private readonly IService _service = service; // Assign within initialization

    // Neither of these methods alters the IService dependency 
    // but each uses a different reference to it
    [HttpGet]
    public ActionResult<Distance> GetOtherDistance()
    {
        return _service.GetDistance();
    }

    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return service.GetDistance(); 
    }
}
```
{: .nolineno }

In this case, now the class has two different references to the same thing. One mutable, one immutable. This is just confusing
and is not recommended either.

### Primary constructor parameters don't become properties, except in record types
This one is wildly important. 

## Conciseness vs consistency vs shallow immutability
One selling point of primary constructors is conciseness. It's cleaner to just have constructor parameters in class declaration
instead of creating a separate constructor method. 

```csharp
// This is much cleaner, easier to read, and contains fewer lines
public class ExampleController(
    IService service, 
    IAnotherService anotherService, 
    IOptions<SomeConfig> options) : ControllerBase
{
    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return service.GetDistance(); 
    }
}

// Traditional constructor with assigning to readonly fields
public class ExampleController : ControllerBase
{
    private readonly IService _service;
    private readonly IAnotherService _anotherService;
    private readonly IOptions<SomeConfig> _options;

    public ExampleController(
        IService service, 
        IAnotherService anotherService, 
        IOptions<SomeConfig> options)
    {
        _service = service;
        _anotherService = anotherService;
        _options = options;
    }

    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return _service.GetDistance(); 
    }
}
```

But what if another class, with its own dependencies, requires a specific implementation that a dependency cannot fulfill?
Let's consider a configuration object that is static and cannot hot reload, so it's injected with `IOptions`. The configuration
object it contains is a simple collection of strings.

```csharp
public class SimpleConfig
{
    public ICollection<string> ImportantStrings { get; set; }
}
```
{: .nolineno }

A new provider called `MyProvider` needs this config object, however, it's important this class does not allow changes to the
configuration within the class itself. It's injected via primary constructor.



```csharp
public class ExampleController(IService service) : ControllerBase
{
    private readonly Imm
    private readonly IService _service = service; // Assign within initialization

    // Neither of these methods alters the IService dependency 
    // but each uses a different reference to it
    [HttpGet]
    public ActionResult<Distance> GetOtherDistance()
    {
        return _service.GetDistance();
    }

    [HttpGet]
    public ActionResult<Distance> Get()
    {
        return service.GetDistance(); 
    }
}
```
{: .nolineno }

## Final Thoughts
Although well intentioned, primary constructors are honestly just best left using with `records`. I think a motivator for this
was that many devs just didn't like having to create readonly fields and assign them in a constructor all the time. But there's 
a very good reason that pattern exists and should continue.

