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
---

When working on software projects that are not side-projects, there are a few patterns I default to. While I try to evolve the implementation of these patterns over time, I find that the core problem they address remains the same, I have just never really found the one with the best balance of simplicity and flexibility. One of these patterns is the `Result` type. In this blog post I will try to explain the problem I am trying to solve, and the latest iteration of my solution. I will give you some background on the problem, and I will show you some of the previous iterations.

## The problem

For most of my software solutions I default to my interpretation of [Hexagonal architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)), I have some plans for a blog-post that will go into that, keep an eye out for that. In that architecture we work with ports and adapters, and we try to keep the core of our application free from any framework or library specific code. The Port describes the expected behavior an adapter needs to comply with. A very simple example of a port could be the following:

```csharp
public interface Users
{
    User ById(UserId id);
}
```

The goal is to push as much if not all implementation details to the adapter. But as you can imagine retrieving a user by a UserId can have a few different outcomes. The user might not exist, the adapter retrieving it might have an implementation error, or the infrastructure dependency the adapter represents might be unavailable. In all these cases we want to be able to steer the business flow and have the system behave in a predictable way. In the case of a "The user might not exist" the business code that we are writing might be perfectly ok with that and just create a new user, or it might need to error out the flow. In that case we could solve it simply by adjusting the interface to the following:

```csharp
public interface Users
{
    User ById(UserId id);
    User? TryById(UserId id);
}
```

Personally I am not a big fan of this solution, it is very explicit, which is nice, but it also increases the complexity of the interface.

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

#### OneOf

Another great NuGet package we came across was [OneOf](https://www.nuget.org/packages/OneOf/). This package is quit a bit more flexible and explicit than the CSharpFunctionalExtensions package. And while the infectious issue we had with CSharpFunctionalExtensions is still there, we found it a lot more manageable.

An implementation of the previous example using OneOf would look something like this:

```csharp
public record Failure;
public record NotFound;

public interface Users
{
    OneOf<User, NotFound, Failure> User ById(UserId id);
}
```

I really like the explicit nature of this implementation, it is a bit verbose, but it is very clear what the interface is trying to communicate.
The consumer of this interface would look something like this:

```csharp
var result = _users.ById(id);
result.Match(
    user => // success, we have a user do something with the user,
    notFound => // handle the not found case,
    failure => // handle the failure case
);
```

### Second Custom implementation

While I quite like the OneOf implementation, I still wanted to try my hand at creating my own implementation.

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

## Conclusion

Did I miss an obvious solution? Something different from a `Result` like type? Do you have feedback on my latest implementation? Please let me know in the comments below.
