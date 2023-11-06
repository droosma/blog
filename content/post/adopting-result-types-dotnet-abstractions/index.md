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

## Introduction

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

```csharp

## The Result type

### Available NuGet packages

[CSharpFunctionalExtensions](https://www.nuget.org/packages/CSharpFunctionalExtensions/)
[OneOf ](https://www.nuget.org/packages/OneOf/)

### Custom implementation version 1

### Custom implementation version 2
