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

Around the internet, you read it more and more: stop testing your database logic with an in-memory database. It's not the same as the real thing. I've agreed with this perspective for quite some time, yet finding a practical way to implement this advice has been somewhat challenging. Since Docker's advent, whenever I needed to test something involving a database, I would gaze at Docker with hope, attempt to utilize it, and then, facing difficulties, revert to either complex real database setups or in-memory alternatives. However, as tooling has evolved, I've reached a point where I can confidently recommend [Testcontainers](https://testcontainers.com/) for testing your database logic.

This article turned out longer than anticipated, so I've divided it into the following sections:

- [**How?**](#how)
  Not familiar with `Testcontainers`? I'll guide you through setting it up and integrating it into your tests.
- [**Optimization**](#optimization)
  The initial motivation for this article was the performance and isolation challenges encountered while using `Testcontainers`. I'll share my strategies for addressing these issues.
- [**Why?**](#why)
  Numerous articles online advise against using in-memory databases for testing, offering plenty of sound arguments. In this section, I'll share my perspective on why I support this view.

All code examples in this post are available [here](https://github.com/droosma/docker-database-testing-journey), utilizing the following tool stack:

- Testing framework: [xUnit](https://xunit.net/)
- Object-Relational Mapping: [Dapper](https://www.learndapper.com/)
- Assertion framework: [FluentAssertions](https://fluentassertions.com/)

All of these can be replaced by other tools; these are just the ones I keep falling back on. If you have suggestions for tooling that you are using and feel is superior, or if you feel I'm missing an important detail, please feel free to comment on the post. I'm always happy to learn new things.

## How?

To simplify the explanation, I will use the following C# [`Repository`](https://www.geeksforgeeks.org/repository-design-pattern/) as my [`System under Test (SUT)`](https://en.wikipedia.org/wiki/System_under_test):

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

This Users repository receives a PostgreSQL connection factory as constructor parameter. It contains a `NameBy` method to query the users table for a user's name based on their ID.

For this example, I'll use a PostgreSQL database via the [`Testcontainers.PostgreSql`](https://testcontainers.com/modules/postgresql/) package. Testcontainers supports a variety of databases through specific packages or by configuring the container manually.

Let's create a test for the `NameBy` method. We begin by setting up a test class to utilize Testcontainer:

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

We initiate a `PostgreSqlContainer` instance, leveraging the `Testcontainers.PostgreSql`'s `ContainerBuilder`. Since starting the Docker image (in this case, PostgreSQL) can take some time—especially on the first run, when the image needs to be downloaded—I use the `IAsyncLifetime` interface for asynchronous setup and teardown of the test lifecycle.

Now, with a running PostgreSQL instance, we can proceed with our test:

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

After completing the `InitializeAsync` method, which starts the PostgreSQL container, we have a connection string to the active PostgreSQL instance. By default, it randomizes the port.

Next, we configure a connection factory as required by our SUT, using a `NpgsqlDataSource` configured with the connection string. This factory is designed to return an open `NpgsqlConnection` via the `OpenConnectionAsync` method upon each invocation.

Since we have a fresh instance of PostgreSQL running in a container, we need to initialize it by creating a users table and inserting a user record. This setup matches the "Arrange" step of the common "Arrange-Act-Assert" testing pattern.

Running this test should result in a pass, demonstrating successful database logic testing with a real database environment.

## <span id="optimization">Optimization</span>

The initial example is functional but becomes inefficient when multiple tests are involved, as it starts the PostgreSQL container for each test. This isn't an issue with a small test suite, but as the number of tests increases, so does the total test execution time. To optimize, we can start the container once for all tests within a test class, using xUnit's `Class Fixtures` for shared context.

Here's how we adapt our `PostgresUsersTests` class:

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

This class can now function as a shared context for our tests, moving some common setup code outside individual test methods. xUnit's `Class Fixtures` alter the test lifecycle, executing `InitializeAsync` once for all tests in the class, rather than on a per-test basis.

We then adapt the `PostgresUsersTests` class to employ `PostgreSQLFixture`:

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

Implementing `IClassFixture<T>`, xUnit injects an instance of `PostgreSQLFixture` into the test class, initializing it before any tests run and disposing of it afterward. However, this introduces potential issues with test isolation, as seen when adding another test that conflicts due to shared database state:

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

Inserting a user with the same ID as in the previous test could lead to a `duplicate key value violates unique constraint "users_pkey"` error.

While introducing transactions and rolling them back after each test could provide the necessary isolation, this method doesn't align with our objective of testing the database logic under real conditions. Transactions that are rolled back don't fully exercise the database's behavior as they would operate in a live environment. This is why, despite the potential for isolation they offer, relying solely on transactions isn't ideal for our purposes. Instead, we leverage xUnit's `Collection Fixtures` for a more suitable approach.

```csharp
[CollectionDefinition(nameof(PostgreSQLFixtureCollection), DisableParallelization = true)]
public class PostgreSQLFixtureCollection : ICollectionFixture<PostgreSQLFixture>
{
}
```

We begin by defining a `PostgreSQLFixtureCollection` class, implementing the `ICollectionFixture<T>` interface. It's annotated with the `CollectionDefinition` attribute, which xUnit uses to identify the shared context for test classes. To ensure tests within this shared context do not run in parallel—thus avoiding isolation problems—we disable parallelization with this attribute. This setup is crucial for maintaining test integrity by preventing concurrent execution that could lead to interference among tests sharing the same database instance.

Finally, we adjust the `PostgresUsersTests` class:

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

By utilizing the `Collection` attribute on our test class, we designate it as part of the shared context defined by the `PostgreSQLFixtureCollection`. This arrangement ensures that all tests within this collection utilize the same `PostgreSQLFixture` instance. Consequently, we only need to initiate the PostgreSQL container once for the entirety of tests in this collection, markedly enhancing performance by eliminating the redundancy of starting the container for each individual test or test class. However, this efficiency necessitates careful management to preserve test isolation and prevent cross-test interference within the shared database instance. Hence, we've reverted to implementing `IAsyncLifetime` for the setup and teardown of database tables specific to each test, thereby removing direct database initialization and disposal responsibilities from the `PostgreSQLFixture`. This adjustment is crucial for upholding isolation among tests.

Despite the effectiveness of this solution, it's not without its drawbacks. I've explored leveraging class inheritance to achieve a similar outcome, aiming for a cleaner or more streamlined implementation. Unfortunately, compatibility challenges with xUnit have hindered these efforts. At present, this approach—combining the Collection attribute with IAsyncLifetime for resource management—stands as our optimal solution. Nevertheless, I'm open to suggestions and keen to explore alternative methods that may offer improved simplicity or efficiency.

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
