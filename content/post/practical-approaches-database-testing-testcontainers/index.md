---
title: "Practical Approaches to Database Testing with Testcontainers"
description: "This blog post explores the benefits of using Testcontainers for database logic testing, highlighting the limitations of in-memory databases and offering insights into optimization techniques for real database testing. It provides a practical guide to integrating Testcontainers with xUnit, Dapper, and FluentAssertions, aiming for more accurate and reliable testing outcomes."
slug: practical-approaches-database-testing-testcontainers
date: 2024-02-24 00:00:00+0000
original: true
image: cover.webp
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

Around the internet, you read it more and more: stop testing your database logic with an in-memory database. It's not the same as the real thing. I've agreed with this perspective for quite some time, yet finding a practical way to implement this advice has been somewhat challenging. Since Docker's advent, whenever I needed to test something involving a database, I would gaze at Docker with hope, attempt to utilize it, and then, facing difficulties, revert to either complex real database setups or in-memory alternatives. However, as tooling has evolved, I've reached a point where I can confidently recommend [`Testcontainers`](https://testcontainers.com/) for testing your database logic.

This article turned out longer than anticipated, so I've divided it into the following sections:

- [**How?**](#how)
  Not familiar with Testcontainers? I'll guide you through setting it up and integrating it into your tests.
- [**Optimization**](#optimization)
  The initial motivation for this article was the performance and isolation challenges encountered while using Testcontainers. I'll share my strategies for addressing these issues.
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

Despite the effectiveness of this solution, it comes with certain drawbacks. My primary concern lies in how the database is reset to a known state. I've considered using class inheritance to achieve a similar outcome, aiming for a cleaner or more streamlined process. However, my experiments with a combination of the `Collection` attribute and `IClassFixture<T>` have yet to meet my expectations. Currently, the approach that combines the `Collection` attribute with `IAsyncLifetime` for resource management represents our best solution. Nonetheless, I remain open to suggestions and am eager to explore alternative methods that might offer greater simplicity or efficiency

### Bonus: Respawn

In my quest to refine the testing setup, I've discovered a promising tool named [`Respawn`](https://github.com/jbogard/Respawn). This package offers an innovative solution for rapid database initialization and cleanup. Unlike traditional methods that involve dropping and recreating databases or tables between tests, Respawn intelligently resets only the data that was added during tests. This approach could significantly reduce the overhead associated with test database management, streamlining the entire testing process.

Although I have not yet successfully implemented Respawn in a production environment, its potential to enhance testing efficiency is evident. I plan to continue experimenting with it and attempting to integrate this tool into my testing workflow. Once I have gathered sufficient insights and experience, I look forward to sharing a detailed follow-up post.

## Closing thoughts

While my experience with Testcontainers has been overwhelmingly positive, it's important to acknowledge feedback from the community indicating occasional instability. I haven't personally encountered these issues, but the concerns merit attention. The reported flakiness may stem from Docker's behavior, particularly on Windows, although similar observations have been made by macOS users as well.

Despite these challenges, the performance trade-offs involved in using Testcontainers are, from my perspective, justified by the confidence it provides in our testing process. The assurance that passing tests correlate with real-world, production-ready code is invaluable. However, this does necessitate careful test design to ensure we're leveraging Testcontainers optimally. It's essential to scope tests accurately, reserving the use of Testcontainers for scenarios where database interactions are critical to the test's validity. This approach helps mitigate performance concerns while maintaining the integrity and relevance of our test suite.

## <span id="why">Why?</span>

I've committed to explaining why in-memory databases may not be the best choice for testing. Having discussed the methodology, let's dive into the rationale behind this perspective.

Given the wide range of opinions on this matter, I'll focus on my experiences and the insights I've gathered from discussions with peers. This approach ensures a narrative grounded in practical encounters and shared viewpoints.

### Encountered Challenges

- **Differences in SQL Dialects**
Encountering discrepancies in SQL dialects, notably with [SQLite](https://system.data.sqlite.org/index.html/doc/trunk/www/index.wiki), has been a recurring issue. These differences often necessitate code alterations for testing purposes, which I view as a significant concern.

- **Transaction Behavior**
Similar to dialect differences, transactional behavior varies significantly. For instance, SQLite's lack of support for nested transactions can limit the scope of tests, affecting their reliability.

- **Data Types and Precision**
Issues arise when SQLite does not support the same data types as the production database, leading to modifications that appear to work until real-world data reveals discrepancies. PostgreSQL's strict enforcement of data precision has been a particular challenge, highlighting the risks of assuming compatibility.

- **Migration and Schema Changes**
Collaborating on a project with a robust database migration workflow, including comprehensive up and down scripts, revealed SQLite's shortcomings. The necessity to devise SQLite-specific migrations and the inability to execute real production migrations presented significant challenges. This prompted a shift to using a real database for testing. However, this transition unveiled numerous issues: migrations that had never been tested for rollback failed, exposing errors that had gone undetected.

- <span id="sense_of_security">**False Sense of Security**</span>
Perhaps the most concerning issue is the false confidence that tests can provide. The differences between testing and production environments mean you might not be thoroughly testing your application's logic, leading to overlooked flaws.

### Considerations Against Discontinuing In-Memory Databases

- **Speed of Execution**
Yes, in-memory databases are quicker than their physical counterparts but can still be slow. If, after implementing the approach I recommend, you find the performance intolerably sluggish, it may be challenging. However, I advise revisiting your testing strategy to confirm it delivers the intended benefits without fostering a [_False Sense of Security_](#sense_of_security).

- **Simplicity and Ease of Configuration**
The argument that in-memory databases simplify testing doesn't sit well with me. Altering code just to facilitate testing feels fundamentally wrong, especially when such modifications diverge from production realities.

- **Resource and Dependency Reduction**
My preference for the [Ports and Adapters](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)) pattern might introduce bias, but testing database-related logic inevitably introduces dependencies, whether in-memory or real. Both approaches demand effort to integrate effectively.

- **Isolation from External Systems**
The idea of complete isolation from external systems is a myth, particularly when a database is integral to production. Ensuring tests accurately reflect real database interactions offers more reliability and consistency in production environments.

- **Avoiding Licensing and Cost Issues**
Licensing and operational costs are legitimate concerns. Ideally, database providers would offer a free tier to mitigate these issues, but this isn't always the case. If costs become prohibitive, it may be worth exploring alternatives.

I'm at a point where, for me, I will need convincing to use an in-memory database for testing, as opposed to the other way around. I'm not saying it's impossible, but by default, I would want to hear a compelling argument for it. Here's a non-exhaustive list of arguments that have swayed me:

- **Rapid Prototyping and Development Cycles**
If the project is in its infancy and the database and its requirements are still evolving, in-memory databases might be a good fit. However, I would still want to transition to a real database as soon as feasible.

- **Unit Testing Specific Database Interactions**
While in-memory databases might not fully mimic the behavior of a production database, they can still be useful for unit testing specific database interactions or data access layers, assuming the tests are designed with the limitations of in-memory databases in mind.
