---
layout: post
title: A few warnings about primary constructors in C#
date: 2025-08-09 17:02 -0600
categories:
- Coding
- C#
tag:
- ".net"
- c#
- design patterns
- style
description: This post goes over the primary constructor feature in C#12 and why you
  may not want to use it. 
---
## TL;DR
Avoid using primary constructors for `class` and `struct` types. Only use them for `record` and `record struct` types because that is
what they were originally intended and best designed for. If you must use them for classes or structs, use them for plain, simple
objects and be very careful about altering the constructor's parameters within the class.

## A brief history and explanation
Primary constructors have been around since C#10 when `record` was introduced. It made perfect sense that records and primary 
constructors came together because of the nature of how `record` works and how they are used. An immutable, data-driven object with 
required parameters to instantiate it? Perfect. 

However, when C#12 introduced primary constructors to `struct` and `class` types, it (heavy air quotes) "simplified" constructors. 
For example, prior to C#12, the structure of the class below may be very familiar. It ensured properties were set when the class was 
constructed by requiring parameters that assigned them within the constructor.

```csharp
public class Customer
{
    public string FirstName { get; }
    public string LastName { get; }

    public Customer(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
    }
}
```
{: .nolineno }

With primary constructors, this same class could be simplified down to something like this instead:

```csharp
public class Customer(string firstName, string lastName)
{
    public string FirstName { get; } = firstName;
    public string LastName { get; } = lastName;
}
```
{: .nolineno }

Perhaps for simple classes, this works and makes sense. It looks clean and is easy to read, but that's about it. In this example, 
one may wonder why not using a `record` instead? If it must be a `class` and you're going to inject dependencies like other services
and add methods, then you'll quickly see some pitfalls of using the primary constructor to facilitate that.

## Microsoft's rules about primary constructors can be misleading
There are a few [rules](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/primary-constructors#understand-rules-for-primary-constructors){:target="_blank"}
that Microsoft states about primary constructors that are somewhat misleading and require additional context. Let's break them down.

> Rule #1: Primary constructor parameters might not be stored if they aren't needed

Basically, if a primary constructor takes a parameter, but doesn't use it, it's not stored. What does this mean exactly?
Consider the following:

```csharp
public class Customer(string firstName, string lastName)
{
    // only firstName is used, so lastName is never part of the compiled object
    public string FirstName { get; } = firstName; 
}

public class Customer(string firstName, string lastName)
{
    // lastName is used, so the compiled object will have it
    public string FullName { get; } = firstName + " " + lastName; 
}
```
{: .nolineno }

In the first `Customer` class, the `lastName` parameter is not used within the class at all. All that Microsoft really means with 
this rule is that the compiler won't store `lastName` within the underlying compiled code. So what? If you require this parameter 
and never use it, who cares what happens with it? Most editors and IDEs would warn you about the first example stating 
that `lastName` is unread and never used. It would be generally a bad practice to have this parameter at all if it's never used anyway.

I think it's an interesting *implementation detail* about how primary constructors work, but is generally not a valuable lesson
to share about this feature.

> Rule #2: Primary constructor parameters aren't members of the class

This is one of the more confusing statements. Yes, parameters in the constructor don't become "members" (heavy air quotes again), 
but all that means is that the *instance* of your class can't directly access it like a member via `this` keyword. 

```csharp
public class Customer(string firstName, string lastName)
{
    public void WriteName()
    {
        // This would not compile, firstName and lastName are not members of the class
        Console.WriteLine(this.firstName + " " + this.lastName);
    }
}
```
{: .nolineno }

This is a good thing. If you never declared a member and could magically access it within the instance of my class, that would be 
unexpected. 

However, this rule is misleading because although primary constructor parameters don't become *members* and they are not properties, 
they are *in scope* of the class. The example above can simply remove references to `this` and access the parameters directly, even
if outside the constructor.

```csharp
public class Customer(string firstName, string lastName)
{
    public void WriteName()
    {
        // This is valid and works because the primary constructor parameters do become in scope of the class
        Console.WriteLine(firstName + " " + lastName);
    }
}
```
{: .nolineno }

There are *serious* implications of this behavior (more on that in Rule #3 below).

> Rule #3: Primary constructor parameters can be assigned to

If there were no other reasons to *not* use primary constructors in classes, this one is convincing enough on its own. There's 
something especially concerning about allowing a parameter to go into scope of a class, and be accessible *throughout* the class, 
*and* be mutable.

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

This is seemingly straightforward and generally makes sense. However, let's consider if *another* method is added to the controller.
But maybe, for some reason, the dev decides it needs a new implementation of `IService` to do this.

```csharp
// New implementation of IService (not registered at startup to inject in .NET DI)
public class OtherDistanceService : IService
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

> Controllers may not be the best example of this due to lifetimes of DI services. A new instance of the controller would be
> created per request, meaning `service` wouldn't change between method calls but if this pattern was applied to any other service,
> the point remains that this could very much negatively that class's behavior.
{: .prompt-warning}

*The fact that parameters in primary constructors are not immutable is one of the biggest reasons I advocate *against* them.*

Now, you may argue a couple of points on this. 

First, even before primary constructors, devs could assign an injected dependency to
a private field that is *not* readonly and encounter the same problem. I agree, but this anti-pattern is far more familiar and easy 
to spot when comparing to primary constructors. 

Second, you could argue that this could be mitigated by making `service` "immutable" 
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
        // _service would be unable to be modified here because it's readonly
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
> keyword this way, especially for reference types, so I would *not* recommend this overall.
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

In this case, now the class has *two different references to the same thing*. One mutable, one immutable. A developer could write a 
new method and use `_service` then another dev comes along with another method and references `service`. This is just confusing
and is not recommended.

> Rule #4: Primary constructor parameters don't become properties, except in record types

This one is is important to point out. In records, parameters in primary constructors become *positional properties*. That serves its
purpose for records, but for classes, this is not the case. There are no explicit getters or setters for parameters passed into
primary constructors and there is no way to modify this behavior (with good reason). However, like as pointed out above, 
you can still modify the parameter within scope of the class and that's not the desired behavior.

## Conciseness vs consistency
One selling point of primary constructors is conciseness. It's cleaner to just have constructor parameters in class declaration
instead of creating a separate constructor method. 

```csharp
// This is much cleaner, easier to read, and contains fewer lines
public class ExampleProvider(
    IService service, 
    IAnotherService anotherService, 
    IOptions<SomeConfig> options) : IExampleProvider
{
    public int Get()
    {
        var data = service.GetData();
        return anotherService.GetIntNeeded(data); 
    }
}

// Traditional constructor with assigning to readonly fields
public class ExampleProvider : IExampleProvider
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

    public int Get()
    {
        var data = _service.GetData();
        return _anotherService.GetIntNeeded(data); 
    }
}
```

But this conciseness may conflict with consistency.

Let's consider a class that only needs a string as a dependency for the class itself. The value of it comes from config and maybe
it's static in nature and is not important to allow hot reloading so it's injected into the class via `IOptions`. 

Consider the following example. It's not ideal in any way, but it's meant to show this scenario.

```csharp
public class SimpleService(IOptions<SomeConfig> options) : ISimpleService
{
    private readonly Regex _regex = new Regex(options.Value.RegexPattern);

    public string GetMatchingString(string input)
    {
        var match = _regex.Match(input);
        return match.Value;
    }
}
```
{: .nolineno }

This is straightforward. The options provides the regex pattern for the `Regex` object used in the class. However, if the class
has no need to use `IOptions<SomeConfig>` in no other places in the class, then what purpose does it serve just because it's in
the primary constructor? This example is how it would typically be written:

```csharp
public class SimpleService : ISimpleService
{
    private readonly Regex _regex;

    public SimpleService(IOptions<SomeConfig> options)
    {
        _regex = new Regex(options.Value.RegexPattern);
    }

    public string GetMatchingString(string input)
    {
        var match = _regex.Match(input);
        return match.Value;
    }
}
```
{: .nolineno }

If this class is written in this manner, but then other classes are written with primary constructors, then you end up with 
*consistency* issues. Personally speaking, I'd find it somewhat distracting and would rather just keep primary constructors out
of classes.


## Final Thoughts
Although well intentioned, primary constructors are honestly best left for use with `records` and an argument could be made
to avoid using them there as well. I think a motivator for this feature was that many devs just didn't like having to create 
readonly fields and assigning them in a constructor all the time. It looks cleaner,
but overall it opens up so many pitfalls by introducing unexpected behaviors and bad practices to classes that it's honestly the best
approach to avoid using them. As all things in development, there is no perfect design and trade offs exist everywhere, but hopefully
this provides some warning about this C# feature.
