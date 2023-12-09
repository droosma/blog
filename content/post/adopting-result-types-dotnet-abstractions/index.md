---
title: Adopting Result Types for More Robust .NET Abstractions
description: Discover the elegance of Result Types in .NET and how they revolutionize error handling, leading to more robust and maintainable code abstractions.
slug: adopting-result-types-dotnet-abstractions
date: 2023-11-05 00:00:00+0000
original: true
image: cover.png
categories:
    - Development
tags:
    - dotnet
    - DDD
    - Programming construct
---

When working on software projects destined for production, I often revisit strategies that have proven successful in the past. Among these, the Ports and Adapters pattern, also known as [Hexagonal architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)), stands out. In acknowledging the unpredictable nature of system evolution, I try to limit the blast radius of system changes by envisioning each port as an API gateway to a ["black-box"](https://en.wikipedia.org/wiki/Black_box) system. With this mindset, another awareness comes to the forefront—[Fallacies of Distributed Computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing), particularly the first: _The network is reliable._ By combining these concepts, I aim to alert the consuming code to the possibility of failure without revealing too much implementation detail. This post will outline my journey through the various solutions I've implemented to address this architectural challenge.

## The problem

Let's start with a simple port example. We have a port called `Users` that allows us to retrieve a user by a `UserId`. The interface would look something like this:

```csharp
public interface Users
{
    User By(UserId id);
}
```

The goal we set for ourselves is to convey to the consumer as much information as possible, without leaking implementation details. Looking at this port a consumer can make a few assumptions, if I give it a `UserId` I will receive a `User`, but what happens when I give it a `UserId` that does not exist? Or what happens when "the network is not reliable"? Constraining myself to only reading this interface, and treating it as a black-box, The consumer of this interface has no way of knowing what can or will happen.

The first thing we can do is make the interface a bit more explicit, and add a `Try` method to the interface. This would look something like this:

```csharp
public interface Users
{
    User By(UserId id);
    User? TryBy(UserId id);
}
```

This addresses the: "but what happens when I give it a `UserId` that does not exist?" question, at the cost of making the interface a bit more complex, and assuming that the consumer uses the correct method, in the correct context. But it does not address the: "what happens when the network is not reliable?" question.

@@@ 
the flow from "the problem" to "possible solutions" feels off, I'm looking for either integrate the possible solutions section into the problem section. Or try to expand on the problem section. and flow into possible solutions from there.
@@@

## Possible solutions

As I mentioned in the introduction, I have tried a few different solutions to this problem. I will try to explain them here, and what I liked and disliked about them.

### Exception based flow control

A solution that is often used is to throw an exception. Let's take the previous example of the first interface and implement it with an exception based flow control.

```csharp
internal class PostgreSQLUsers : Users
{
    public User ById(UserId id)
    {
        try
        {
            using (IDbConnection db = new NpgsqlConnection(_connectionString))
            {
                return db.Query<User>("SELECT * FROM users WHERE id = @Id", new { Id = id.Value }).SingleOrDefault() ?? throw new UserNotFoundException(id);
            }
        }
        catch (NpgsqlException ex)
        {
            throw new ApplicationException("An error occurred while accessing the database.", ex);
        }
        catch (Exception ex)
        {
            throw;
        }
    }
}
```

#### Benefits

- We can easily expand the problem space by creating more specific exceptions.
- Centralized error handling, we can filter certain exceptions and handle them in a centralized and consisted way.

#### Drawbacks

- Exceptions are expensive, they are not meant to be used for flow control. Exceptions are meant to be exceptional, and the fact that a user might not exist is not exceptional.
- Exceptions are not very explicit, the interface does not tell us that it might throw an exception.
- Exceptions are not very flexible, we can not easily change the behavior of the system based on the exception. Given our example of having a business flow that can handle a user not existing, we would need to wrap the call to the `Users` interface in a `try/catch` and handle the exception there.
- They are leaky, it is very easy to "leak" PostgreSQL specific implementation details to the calling code.

### Null based flow control

Another solution that is often used is to return `null` when the user does not exist. Let's take the previous example of the first interface and implement it with a null based flow control.

```csharp
internal class PostgreSQLUsers : Users
{
    public User? ById(UserId id)
    {
        try
        {
            using (IDbConnection db = new NpgsqlConnection(_connectionString))
            {
                return db.Query<User>("SELECT * FROM users WHERE id = @Id", new { Id = id.Value }).SingleOrDefault();
            }
        }
        catch (NpgsqlException ex)
        {
            throw new ApplicationException("An error occurred while accessing the database.", ex);
        }
        catch (Exception ex)
        {
            throw;
        }
    }
}
```

The way I see it, the only thing this approach brings to the table is that it is explicit about the fact that a function might return nothing, so we could get away with not having a `User? TryById(UserId id);`. But it does not solve any of the other problems.

### Callback based flow control

### Result based flow control

A way to make the flow very explicit is to use a `Result` type. A `Result` type is a type that can have two different states, a success state and a failure state. This immidatly communicates to all consumers of the code that the function might fail.

## The Result type

### Available NuGet packages

#### CSharpFunctionalExtensions

A few years ago I came across a NuGet package called [CSharpFunctionalExtensions](https://www.nuget.org/packages/CSharpFunctionalExtensions/). This package contains a `Result` type that can be used to solve the problem we are trying to solve. The package is very easy to use, and it has a lot of other useful types and functions. I have used this package in a few projects, and I have been very happy with it. But I have also found that it suffers from the `async/await` problem

In dotnet we run into a infectious solution that changes all implementations in the chain, like async and await. An beautiful solution to asynchronicity but, every piece of code in the chain will be effected by it. The same goes for the Result or Maybe types in CSharpFunctionalExtensions. It's a brilliant and clean functional way to deal with the problem. But the way you consume the code means your entire pipeline starts to move to an functional implementation. This can increase the cognitive load on developers not familiar with functional development.

An implementation of the previous example using `Maybe` from CSharpFunctionalExtensions would look something like this:

```csharp
public interface Users
{
    Result<Maybe<User>> ById(UserId id);
}
```

The consumer of this interface would look something like this:

```csharp
var result = _users.ById(id);

return result.IsSuccess 
                   ? result.Value.HasValue 
                         ? Ok(result.Value.Value) // Some(User) case
                         : NotFound() // None case
                   : StatusCode(StatusCodes.Status500InternalServerError);
```

### First Custom implementation

Dealing with the infectious nature of the CSharpFunctionalExtensions package, and the resulting complexity in our software, we decided to try our hand in creating our own implementation of the Result type. But just the parts we needed. The implementation looked something like this:

```csharp
public record Result
{
    public virtual bool IsSuccess { get; }
    public virtual bool IsFailure => !IsSuccess;

    protected Result(bool isSuccess)
    {
        IsSuccess = isSuccess;
    }

    public static Result Success => new (true);
    public static Result Failed => new Result(false);
}

public record Result<T> : Result
{
    private bool _isChecked = false;
    private readonly T _value;
    
    public T Value
    {
        get
        {
            if(!_isChecked)
                throw new ValueNotVerifiedException();
            return _value;
        }
    }

    public override bool IsSuccess
    {
        get
        {
            _isChecked = true;
            return base.IsSuccess;
        }
    }

    public static implicit operator T(Result<T> result)
        => result.Value;
    
    public override bool IsFailure => !IsSuccess;

    protected Result(T value, bool isSuccess) : base(isSuccess)
    {
        _value = value;
    }
    
    [Serializable]
    public sealed class ValueNotVerifiedException : Exception
    {
        public ValueNotVerifiedException()
        {
        }

        private ValueNotVerifiedException(SerializationInfo info, StreamingContext context) : base(info, context)
        {
        }
    }
}
```

With this implementation we found that sometimes people would still assume success and use the Value property without checking the IsSuccess or IsFailure property. So to prevent runtime errors we added the ValueNotVerifiedException. This confronted a developer immediately during initial run of the code, usually a unit test. This implementation worked well for us for a while, but the implementation is not very flexible. Adding new states to the Result type resulted in a lot of ugly implementation refactoring.

### Second Custom implementation

A second attempt at a custom implementation of the Result type inspired by some of the Discriminated Union patterns, is a bit more flexible.

```csharp
public interface Result
{
    public static Result Succeeded() => new Succeeded();
    public static Result Failed(Exception? exception = null) => new Failed(exception);
}

public interface Result<out T> : Result
{
    public T Value { get; }

    public static Result<T> Succeeded(T value) => new Succeeded<T>(value);
    public new static Result<T> Failed(Exception? exception = null) => new Failed<T>(exception);
}

public record Succeeded : Result;
public record Succeeded<T>(T Value) : Succeeded, Result<T>;

public record Failed : Result;
public record Failed<T> : Failed, Result<T>
{
    public T Value => throw new Exception("Failed result does not have a value");
}
```

This implementation is not very battle hardened quite yet

#### OneOf

As it often is the case, I'm not the only one trying to solve this problem. And there is always someone smarter than me that has already solved it. As is the case with [OneOf](https://www.nuget.org/packages/OneOf/). This package is quit a bit more flexible and explicit than the CSharpFunctionalExtensions package.

An implementation of the previous example using OneOf would look something like this:

```csharp
public record Failure;
public record NotFound;

public interface Users
{
    OneOf<User, NotFound, Failure> User ById(UserId id);
}
```

I really like the explicit nature of this implementation, it is a bit verbose, but it is very clear what the interface is trying to communicate. The packages also introduces a `Match` function that allows you to handle the different cases in a very clean and explicit way. Using the interface would look something like this:

```csharp
var result = _users.ById(id);
result.Match(
    user => // success, we have a user do something with the user,
    notFound => // handle the not found case,
    failure => // handle the failure case
);
```

## Conclusion

Did I miss an obvious solution? Something different from a `Result` like type? Do you have feedback on my latest implementation? Please let me know in the comments below.
