---
title: "Practical Approaches to Database Testing with Testcontainers"
description: "This blog post explores the benefits of using Testcontainers for database logic testing, highlighting the limitations of in-memory databases and offering insights into optimization techniques for real database testing. It provides a practical guide to integrating Testcontainers with xUnit, Dapper, and FluentAssertions, aiming for more accurate and reliable testing outcomes."
slug: practical-approaches-database-testing-testcontainers
date: 2024-02-24 00:00:00+0000
original: true
image: cover.png
categories:
    - Development
tags:
    - dotnet
    - docker
    - testing
    - xUnit
    - Testcontainers
    - PostgreSQL
---

You hear it more and more, stop testing you database logic with an in-memory database. It's not the same as the real thing. And I have agreed with this for a long time. But actually find a way to do it has been a bit of a process. Since the advent of Docker, every time I had to test something with a database, I would look longingly at it, try to make it work, and then give up and resort to either complicated real database setups or in-memory versions. But with time, tooling has come a long way and, and I'm finally at a point to start recommending [`Testcontainers`](https://testcontainers.com/) for testing your database logic.

This article grew a bit longer than I expected, so I have split it up into the following sections:

- [How?](#how)
  Not familiar with `Testcontainers`? I'll show you how to set it up and use it in your tests.
- [Optimization](#optimization)
  The original reason I started writing this article, when using `Testcontainers` you'll run into some performance issues, and some isolation issues. Here I'll show you my approach to solving these issues.
- [Why?](#why)
  There are lot's are articles online that will tell you to stop using in-memory databases for testing, so lot's of good content out there. In this section I have tried to give you my reasoning for why I think you should.

All the code used in this post can be found [here](https://github.com/droosma/docker-database-testing-journey), where I'm using the following tool stack:

- Testing framework: [xUnit](https://xunit.net/)
- [Object-Relational Mapping](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping): [Dapper](https://www.learndapper.com/)
- Assertion framework: [FluentAssertions](https://fluentassertions.com/)

All of these can be replaced by other tools, these are just the once I keep falling back on. If you have suggestions for tooling that you are using and feel is superior, or you feel I'm missing an important detail, please feel free to comment on the post. Always happy to learn new things.

## How?

To keep the explanation as simple as I could get it, I'm going to be using the following C# [`Repository`](https://www.geeksforgeeks.org/repository-design-pattern/) as my [`System under Test (SUT)`](https://en.wikipedia.org/wiki/System_under_test)

```csharp
public class PostgresUsers(Func<ValueTask<NpgsqlConnection>> connectionFactory)
{
    public async Task<string?> NameBy(int userId)
    {
        await using var connection = await connectionFactory();
        return await connection.QueryFirstOrDefaultAsync<string>("SELECT name FROM users WHERE id = @Id", new {Id = userId});
    }
}
```

A simple Users repository that gets a PostgreSQL connection factory injected, with a `NameBy` method that queries the users table for the name of a user with a given id.

In this example I'm going to use a PostgreSQL database though the [`Testcontainers.PostgreSql`](https://testcontainers.com/modules/postgresql/) package, but there are loads of other databases supported by `Testcontainers` though specific packages or just by configuring the container yourself.

Let's write a test for `NameBy` method. We start by configuring a test class to use the `Testcontainer`

```csharp
public class PostgresUsersTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder().Build();

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
    }

    public Task DisposeAsync() => _postgres.DisposeAsync().AsTask();
}
```

We start with creating an instance of `PostgreSqlContainer`, a wrapper around the `ContainerBuilder` from `Testcontainer` provided by `Testcontainers.PostgreSql`. As the startup of the docker image of in this case `PostgreSQL` can take a little while, especially if it's the first time running this image and it still needing to be pulled from the image repository. I'm using the `IAsyncLifetime` interface to facilitate the async setup and teardown test lifecycle.

Now that we have a running `PostgreSQL` instance, we can write our test.

```csharp
[Fact]
public async Task NameBy_WhenUserExists_ReturnsName()
{
    var connectionString = _postgres.GetConnectionString();
    
    var dataSource = new NpgsqlDataSourceBuilder(connectionString).Build();
    var connectionFactory = () => dataSource.OpenConnectionAsync();

    await using var connection = await connectionFactory();
    await connection.ExecuteAsync("CREATE TABLE users (id BIGSERIAL PRIMARY KEY, name TEXT)");
    await connection.ExecuteAsync("INSERT INTO users (id, name) VALUES (1, 'John Doe')");
    
    var sut = new PostgresUsers(connectionFactory);
    var name = await sut.NameBy(1);
    
    name.Should().Be("John Doe");
}
```

Now that we are in a test method, xUnit will have completed the `InitializeAsync` which started the `PostgreSQL` container. Which should give us access to the connection string of the running `PostgreSQL` instance, while you can configure it, by default it randomizes the port.

Next we create an connection factory the way our SUT expects it by configuring a `NpgsqlDataSource` with the connection string and then configure the factory to return a open `NpgsqlConnection` with the `OpenConnectionAsync` method every time the factory is invoked.

Seeing as we still only have a database container running a fresh and clean instance of `PostgreSQL`, we need to initialize it by creating a table and insert a user. We do this by creating a connection with the connection factory and then executing the SQL to create the table and insert a user that should match our `Act` of the often used `Arrange-Act-Assert` test pattern.

If we run the test now, we should see it pass. And we have successfully tested our database logic with a real database.

## <span id="optimization">Optimization</span>

The above example will work, but fall's apart when you have more than one test. As the `PostgreSQL` container is started for every test. This is not a problem for a small amount of tests, but as the amount of tests grow, the time it takes to run them will grow as well. So we can optimize this by starting the container once for all tests in the test class. xUnit has a feature for shared context called `Class Fixtures` that we can use for this.

We take the original `PostgresUsersTests` class and change it to the following:

```csharp
public class PostgreSQLFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder().Build();
    public Func<ValueTask<NpgsqlConnection>> ConnectionFactory = default!;
    public string ConnectionString { get; private set; } = default!;

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        ConnectionString = _postgres.GetConnectionString();
        var dataSource = new NpgsqlDataSourceBuilder(ConnectionString).Build();
        ConnectionFactory = () => dataSource.OpenConnectionAsync();

        await using var connection = await connectionFactory();
        await connection.ExecuteAsync("CREATE TABLE users (id BIGSERIAL PRIMARY KEY, name TEXT)");
    }

    public Task DisposeAsync() => _postgres.DisposeAsync().AsTask();
}
```

As this class is going to function as our tests shared context, we have moved out some of the boilerplate code from the test, `Class Fixtures` feature change the way xUnit's test lifecycle works. It change the default behavior of calling `InitializeAsync` on the test class once for each test, it will now only execute this once for all the tests.

We can now change the `PostgresUsersTests` class to use the `PostgreSQLFixture` as a shared context.

```csharp
public class PostgresUsersTests(PostgreSQLFixture fixture) : IClassFixture<PostgreSQLFixture>
{
    [Fact]
    public async Task NameBy_WhenUserExists_ReturnsName()
    {
        await using var connection = await fixture.ConnectionFactory();
        await connection.ExecuteAsync("INSERT INTO users (id, name) VALUES (1, 'John Doe')");

        var sut = new PostgresUsers(fixture.ConnectionFactory);
        var name = await sut.NameBy(1);
        
        name.Should().Be("John Doe");
    }
}
```

By implementing the `IClassFixture<T>` interface xUnit will inject an instance of `PostgreSQLFixture` into the test class. This instance will be initialized before starting execution of any of the tests inside the test class and disposed after all the tests have been executed. Using this approach introduces a small problem though, as the `PostgreSQLFixture` is shared between all the tests in the test class, it also means that our database tests are no longer isolated from each other. We can see this in action by adding another test to the `PostgresUsersTests` class.

```csharp
[Fact]
public async Task NameBy_WhenUserDoesNotExists_ReturnsNull()
{
    await using var connection = await fixture.ConnectionFactory();
    await connection.ExecuteAsync("INSERT INTO users (id, name) VALUES (1, 'John Doe')");

    var sut = new PostgresUsers(fixture.ConnectionFactory);
    var name = await sut.NameBy(2);

    name.Should().BeNull();
}
```

By inserting a user with the same id as the previous test, we can see that the second test will fail with a unique constraint violation.

We can work around this by introducing a `Transaction` and just rolling back the transaction after each test. This will give us the isolation we need, but the reason we are using a real database is to test the actual behavior of our database logic. So we can't just roll back the transaction, as we would not be testing the actual behavior of our database logic. We can however use another feature of xUnit, `Collection Fixtures`.

```csharp
[CollectionDefinition(nameof(PostgreSQLFixtureCollection), DisableParallelization = true)]
public class PostgreSQLFixtureCollection : ICollectionFixture<PostgreSQLFixture>
{
}
```

We start by creating a `PostgreSQLFixtureCollection` class that implements the `ICollectionFixture<T>` interface. We mark it with the `CollectionDefinition` attribute a marker used by xUnit to identify which shared context to use for a test class. We also disable parallelization, as we don't want tests that use the same shared context to run in parallel, as this will cause the isolation issue we experienced.

Next we change the `PostgresUsersTests` class one more time to the following.

```csharp
[Collection(nameof(PostgreSQLFixtureCollection))]
public class PostgresUsersTests(PostgreSQLFixture fixture) : IAsyncLifetime
{
    public async Task InitializeAsync()
    {
        await using var connection = await fixture.ConnectionFactory();
        await connection.ExecuteAsync("CREATE TABLE users (id BIGSERIAL PRIMARY KEY, name TEXT)");
    }

    public async Task DisposeAsync()
    {
        await using var connection = await fixture.ConnectionFactory();
        await connection.ExecuteAsync("DROP TABLE users");
    }

    [Fact]
    public async Task NameBy_WhenUserExists_ReturnsName()
    {
        await using var connection = await fixture.ConnectionFactory();
        await connection.ExecuteAsync("INSERT INTO users (id, name) VALUES (1, 'John Doe')");

        var sut = new PostgresUsers(fixture.ConnectionFactory);
        var name = await sut.NameBy(1);
        
        name.Should().Be("John Doe");
    }
}
```

We start by adding the newly created `Collection` attribute to the test class, and you might have noticed, we have taken a step back, as we are back to using `IAsyncLifetime` this time we use it to initialize and dispose the table we are using for our tests, as we still want to have isolation between our tests. We can now remove this database initialization from the `PostgreSQLFixture`.

I still don't really like this part, I have experimented with using class inheritance to get the same behavior, but xUnit doesn't want to play nice with that. So for now this is the best I have come up with, please let me know if you have a better solution.

Something that I have not mentioned is that before we started using this `ICollectionFixture<T>` feature, we would still create a new database instance for each test class. And assuming you are using a database for more than a single table, this would still degrade performance in the long run. Now that we have this `[Collection(nameof(PostgreSQLFixtureCollection))]` each test class that uses this collection attribute will share the same `PostgreSQLFixture` instance. This means that we only start the `PostgreSQL` container once for all the tests that use this collection. This is a huge performance improvement.

### Bonus: Respawn

In my attempt to streamline the setup of the tests, I have found a package that can help with a faster initialization and cleanup of the database [`Respawn`](https://github.com/jbogard/Respawn) I have not actually used it in production, but the gist of it is that it will reset the data in the database intelligently, so instead of dropping the databases / tables and re-creating them between tests you can just reset the data that has been created.

When I have more experience with it, I will write a follow-up post about it.

## Closing thoughts

While I have been very happy with `Testcontainers` I have heard from others that it can be a bit flaky. I have not experienced this myself but it is worth mentioning. I suspect the flakiness comes from the relative instability of Docker, in my case on Windows, but I hear similar stories from people using it on Mac.

Even with these optimizations, performance is still a thing. While I find the compromise worth it for the peace of mind that I get from knowing when it's green, it's actually green. And the code will work in production. It does require a bit of extra attention to make sure you are scoping your tests correctly and not using it when behavior from the database is not relevant for the test assertion.

## <span id="why">Why?</span>

I promised to explain why I think you should stop using in-memory databases for testing, and now that we go the how out of the way, I'll try to give you some insight into my reasoning.

There are a lot of opinions on this subject, so I'm going to contain myself to my own experiences and the arguments I have heard from others.

### Reasons that I have run into

- **Differences in SQL Dialects**
I have run into this a few times. When using something like [SQLite](https://system.data.sqlite.org/index.html/doc/trunk/www/index.wiki) there are some differences in the SQL dialect that make it impossible to test your logic without changing your code. And for me that's a red-flag if I ever saw one.
- **Transaction Behavior**
This is quite similar to the previous point, but I feel it important to mention separately. As transactional behavior might be outside of the scope of a single query or method you are testing, SQLite does not support nested transactions for example.
- **Data Types and Precision**
Have had this happen a few times, the SQLite database is created but as it does not support the same data types as the database you are using in production, you change the column type to the closes match and move on. You test your logic and everything seems fine. Running it in production and see the exceptions roll in. PostgreSQL is enforcing the precision of the column, and you are receiving data that does not fit in the column.
- **Migration and Schema Changes**
Working with a team that has a clean workflow for database migration, including up and down scripts. But given SQLite's limitations, had to introduce two problems, one write SQLite specific migrations and two, could not run the actual production migrations. So I changed the tests to use a real database, these migrations exploded. They never ran the down migrations and there were errors in it.
- <span id="sense_of_security">**False Sense of Security**</span>
This is one of the most scary ones, as it's very hard to detect and counteract. You are writing tests, you might catch a mistake or two, but as examples above show, you are not testing the same logic as you are using in production. So you are not getting the same coverage as you are hoping for.

### Arguments against it

- **Speed of Execution**
Yes, in memory databases are faster than real databases, still slow though. And after you have tried the approach I'm advocating here, and still find them insufferably slow. Than it's not going to be easy, but I would suggest taking a step back from your tests and making sure they are really giving you what you are hoping for. See [_False Sense of Security_](#sense_of_security).
- **Simplicity and Ease of Configuration**
I have always found this a bit of a weird argument, when ever I need to change code to be able to test it, I feel dirty. And working with an in-memory database is always a little different than the way a real database works, always having to change things to make it work.
- **Resource and Dependency Reduction**
This is probably my observation bias. But I use [Ports and Adapters](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)) a lot. So when I'm testing logic surrounding my persistence I always have to have a database dependency. Both In-Memory and real represent resources and dependencies, getting either to work requires effort.
- **Isolation from External Systems**
It's not like your system is actually isolated from the external system. You are using a database in production. Acting like the database is always there and always working is not going to help you. I would argue getting your tests to behave consistency against a real database is going to help you do the same in production.
- **Avoiding Licensing and Cost Issues**
This is a valid point, although I personally feel every database provider needs to have a free tier for specifically this reason, reality is that this is not always the case. The smart ass in me would say, time to look at a different database provider 😉.
