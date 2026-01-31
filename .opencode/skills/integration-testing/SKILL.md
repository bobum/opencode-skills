---
name: integration-testing
description: Integration testing patterns using DatabaseTestFixture with SQLite in-memory. Use when writing tests that need a real database context in .NET/EF Core projects.
---

# Integration Testing

This skill covers patterns for integration testing with a `DatabaseTestFixture` and SQLite in-memory database.

## DatabaseTestFixture

The fixture provides:
- SQLite in-memory database (real SQL behavior)
- Full DI container with all services
- Seeded test data
- Automatic cleanup

### Fixture Structure

```csharp
public class DatabaseTestFixture : IDisposable
{
    private readonly SqliteConnection _connection;
    public ServiceProvider ServiceProvider { get; private set; }
    public YourDbContext DbContext { get; private set; }

    public DatabaseTestFixture()
    {
        // SQLite in-memory for real SQL behavior
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var services = new ServiceCollection();

        // DbContext with SQLite
        services.AddDbContext<YourDbContext>(options =>
            options.UseSqlite(_connection), ServiceLifetime.Singleton);

        // Register ALL repositories
        services.AddScoped<ILeagueRepository, LeagueRepository>();
        services.AddScoped<ITeamRepository, TeamRepository>();
        // ... all repositories

        // Register ALL services
        services.AddScoped<ITeamBuilderService, TeamBuilderService>();
        // ... all services

        // External dependencies / engines
        services.AddSingleton<IExternalService, ExternalService>();

        ServiceProvider = services.BuildServiceProvider();
        DbContext = ServiceProvider.GetRequiredService<YourDbContext>();

        // Create schema (not Migrate - cross-provider compatible)
        DbContext.Database.EnsureCreated();

        // Seed test data
        SeedDatabase().GetAwaiter().GetResult();
    }

    public void Dispose()
    {
        _connection?.Close();
        _connection?.Dispose();
    }
}
```

---

## Writing Integration Tests

### Basic Test

```csharp
public class TeamServiceTests : IClassFixture<DatabaseTestFixture>
{
    private readonly DatabaseTestFixture _fixture;

    public TeamServiceTests(DatabaseTestFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task CreateTeam_WithValidData_ReturnsTeam()
    {
        // Arrange
        var service = _fixture.ServiceProvider.GetRequiredService<ITeamBuilderService>();

        // Act
        var result = await service.BuildTeamAsync(new TeamBuildRequest { Name = "Test Team" });

        // Assert
        Assert.NotNull(result);
        Assert.Equal("Test Team", result.Name);
    }
}
```

### Testing with Database

```csharp
[Fact]
public async Task GetTeam_WhenExists_ReturnsTeam()
{
    // Arrange
    var repository = _fixture.ServiceProvider.GetRequiredService<ITeamRepository>();

    // Act
    var team = await repository.GetByIdAsync(1);  // Seeded data

    // Assert
    Assert.NotNull(team);
    Assert.Equal("Test Team", team.Name);
}

[Fact]
public async Task SoftDelete_MarksAsDeleted()
{
    // Arrange
    var repository = _fixture.ServiceProvider.GetRequiredService<ITeamRepository>();

    // Act
    await repository.SoftDeleteAsync(2, "test", "testing delete");

    // Assert
    var team = await _fixture.DbContext.Teams
        .IgnoreQueryFilters()
        .FirstAsync(t => t.Id == 2);

    Assert.True(team.IsDeleted);
    Assert.Equal("test", team.DeletedBy);
}
```

---

## Adding New Services to Tests

When adding a new service, you **MUST** update `DatabaseTestFixture.cs`:

```csharp
// In DatabaseTestFixture constructor

// 1. Add repository registration
services.AddScoped<INewFeatureRepository, NewFeatureRepository>();

// 2. Add service registration
services.AddScoped<INewFeatureService, NewFeatureService>();
```

**Common mistake**: Adding a service to `Program.cs` but forgetting `DatabaseTestFixture.cs`. Tests will fail with DI errors.

---

## Test Data Seeding

The fixture seeds baseline data in `SeedDatabase()`:

```csharp
private async Task SeedDatabase()
{
    // Seed leagues
    DbContext.Leagues.Add(new League { Id = 1, Name = "Test League" });

    // Seed entities
    DbContext.Teams.AddRange(
        new Team { Id = 1, LeagueId = 1, Name = "Test Team" },
        new Team { Id = 2, LeagueId = 1, Name = "Another Team" }
    );

    // Seed players, etc.

    await DbContext.SaveChangesAsync();
}
```

### Adding Test Data

For specific tests, add data within the test:

```csharp
[Fact]
public async Task GetPlayersByTeam_ReturnsCorrectPlayers()
{
    // Arrange - add specific test data
    _fixture.DbContext.Players.Add(new Player
    {
        TeamId = 1,
        FirstName = "Test",
        LastName = "Player",
        Position = Position.QB
    });
    await _fixture.DbContext.SaveChangesAsync();

    // Act
    var service = _fixture.ServiceProvider.GetRequiredService<IPlayerService>();
    var players = await service.GetByTeamAsync(1);

    // Assert
    Assert.Contains(players, p => p.LastName == "Player");
}
```

---

## Testing Patterns

### Testing Authorization

```csharp
[Fact]
public async Task UpdateTeam_WhenNotOwner_ThrowsForbidden()
{
    var service = _fixture.ServiceProvider.GetRequiredService<ITeamService>();

    await Assert.ThrowsAsync<ForbiddenException>(() =>
        service.UpdateAsync(1, "wrong-user-id", new TeamUpdateRequest()));
}
```

### Testing Soft Delete

```csharp
[Fact]
public async Task GetAll_ExcludesDeleted()
{
    // Arrange
    var repository = _fixture.ServiceProvider.GetRequiredService<ITeamRepository>();
    await repository.SoftDeleteAsync(1, "test", "test");

    // Act
    var teams = await repository.GetAllAsync();

    // Assert
    Assert.DoesNotContain(teams, t => t.Id == 1);
}
```

### Testing Transactions

```csharp
[Fact]
public async Task CreateLeagueWithTeams_RollsBackOnError()
{
    var service = _fixture.ServiceProvider.GetRequiredService<ILeagueService>();

    // Act - this should fail and roll back
    await Assert.ThrowsAsync<ValidationException>(() =>
        service.CreateWithTeamsAsync(new CreateLeagueRequest
        {
            Name = "Test",
            Teams = new[] { /* invalid team */ }
        }));

    // Assert - no partial data
    var leagues = await _fixture.DbContext.Leagues.CountAsync();
    Assert.Equal(1, leagues);  // Only seeded league
}
```

---

## Running Tests

```bash
# Run all integration tests
dotnet test YourProject.IntegrationTests

# Run specific test class
dotnet test --filter "FullyQualifiedName~TeamServiceTests"

# Run with verbose output
dotnet test --logger "console;verbosity=detailed"
```

---

## Test Coverage Requirements

> **Write thorough tests from the start. Don't "add tests later."**

### Coverage by Layer (Verify as you implement)

| Layer | What to Test | Example |
|-------|--------------|---------|
| Repository → DB | CRUD operations hit real SQLite | `GetByIdAsync` returns seeded entity |
| Service → Repository | Business logic with real data access | Validation + save + retrieve |
| Controller → Service | Full request/response cycle | HTTP 200/400/404 responses |

### Edge Cases (Include these in EVERY test class)

- [ ] Null/empty inputs → appropriate error
- [ ] Entity not found → 404 or null
- [ ] Duplicate creation → constraint violation handled
- [ ] Soft delete → excluded from normal queries
- [ ] Soft delete cascade → related entities marked deleted
- [ ] Concurrent modification → optimistic concurrency handled

### Error Paths (Test these, not just happy paths)

- [ ] Invalid input → validation error returned
- [ ] Missing related entity → appropriate error
- [ ] Database constraint violation → handled gracefully
- [ ] Transaction rollback → no partial state

### Test Isolation (Verify for every test)

- [ ] Test doesn't depend on other tests running first
- [ ] Test cleans up or uses unique data
- [ ] Test would pass if run 100 times in a row
- [ ] Test would pass if run in parallel with others

### Assertions (Make them meaningful)

```csharp
// BAD - just checking no exception
var result = await service.CreateAsync(request);
Assert.NotNull(result);

// GOOD - verifying actual behavior
var result = await service.CreateAsync(request);
Assert.Equal("Expected Name", result.Name);
Assert.True(result.Id > 0);

// Verify database state too
var fromDb = await _fixture.DbContext.Teams.FindAsync(result.Id);
Assert.NotNull(fromDb);
Assert.Equal("Expected Name", fromDb.Name);
```

---

## Checklist for Integration Tests

- [ ] `DatabaseTestFixture` has all required DI registrations
- [ ] Test data seeded for the scenario
- [ ] Happy path tested
- [ ] Error paths tested (not found, validation, constraints)
- [ ] Edge cases tested (null, empty, boundary values)
- [ ] Database state verified after operations
- [ ] Tests are isolated and deterministic
- [ ] Async tests use `async Task`, not `async void`
- [ ] Use `GetRequiredService<T>()` not `GetService<T>()`
