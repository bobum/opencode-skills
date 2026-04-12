---
name: integration-testing
description: Integration testing patterns using DatabaseTestFixture with Postgres Docker. Use when writing tests that need a real database context in .NET/EF Core projects.
---

# Integration Testing

This skill covers patterns for integration testing with `DatabaseTestFixture` and Postgres Docker containers.

## Test Infrastructure

### PostgresTestDatabase

**Location**: `Gridiron.IntegrationTests/PostgresTestDatabase.cs`

Shared helper that manages test database lifecycle:
- Creates uniquely-named databases (`gridiron_it_{guid8}`) to avoid conflicts
- Connection string from env var `TEST_POSTGRES_CONNECTION` or defaults to local Docker (`Host=localhost;Port=5432;Username=gridironadmin;Password=Gridiron@Local123`)
- Retries creation up to 10 times with exponential backoff (1s-8s)
- Cleanup: clears Npgsql connection pool, terminates backends, drops database with `WITH (FORCE)`

```csharp
public sealed class PostgresTestDatabase : IAsyncDisposable
{
    public string ConnectionString { get; }

    public PostgresTestDatabase()
    {
        var dbName = $"gridiron_it_{Guid.NewGuid():N}"[..24];
        // Derives ConnectionString with unique database name
    }

    public async Task EnsureCreatedAsync()
    {
        // Creates the database with retry logic
        // Then applies EF Core EnsureCreated for schema
    }

    public async ValueTask DisposeAsync()
    {
        // Clears connection pool, terminates backends, drops database
    }
}
```

### DatabaseTestFixture

**Location**: `Gridiron.IntegrationTests/DatabaseTestFixture.cs`

```csharp
public class DatabaseTestFixture : IDisposable
{
    private readonly PostgresTestDatabase _postgresTestDatabase;
    public ServiceProvider ServiceProvider { get; private set; }
    public GridironDbContext DbContext { get; private set; }

    public DatabaseTestFixture()
    {
        // Postgres Docker container (shared via docker-compose or CI service)
        _postgresTestDatabase = new PostgresTestDatabase();
        _postgresTestDatabase.EnsureCreatedAsync().GetAwaiter().GetResult();

        var services = new ServiceCollection();

        // DbContext with Npgsql (Postgres)
        services.AddDbContext<GridironDbContext>(options =>
            options.UseNpgsql(_postgresTestDatabase.ConnectionString),
            ServiceLifetime.Singleton);

        // Register ALL repositories
        services.AddScoped<ILeagueRepository, LeagueRepository>();
        services.AddScoped<ITeamRepository, TeamRepository>();
        services.AddScoped<IPlayerRepository, PlayerRepository>();
        // ... register all repositories

        // Register ALL services
        services.AddScoped<ITeamBuilderService, TeamBuilderService>();
        services.AddScoped<IPlayerGeneratorService, PlayerGeneratorService>();
        // ... register all services

        // External engines
        services.AddSingleton<IGameEngine, GameEngine>();
        services.AddSingleton<IInjuryEngine, InjuryEngine>();

        ServiceProvider = services.BuildServiceProvider();
        DbContext = ServiceProvider.GetRequiredService<GridironDbContext>();

        // Create schema (not Migrate - cross-provider compatible)
        DbContext.Database.EnsureCreated();

        // Seed test data
        SeedDatabase().GetAwaiter().GetResult();
    }

    public void Dispose()
    {
        _postgresTestDatabase.DisposeAsync().GetAwaiter().GetResult();
    }
}
```

### Three Fixtures Exist

| Fixture | Purpose | DB per instance? |
|---------|---------|-----------------|
| `DatabaseTestFixture` | Repository-level tests (direct service/repo calls) | Yes |
| `GridironWebApplicationFactory` | HTTP integration tests (full API pipeline) | Yes |
| `InfrastructureTests` | Infrastructure seeding validation | Yes |

Each creates its own Postgres database. On a resource-constrained machine, this compounds startup cost.

### Postgres Docker Setup

**Local**: `docker-compose.yml` runs `postgres:16-alpine` on port 5432.

**CI**: GitHub Actions service container in `.github/workflows/ci-cd.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    env:
      POSTGRES_USER: gridironadmin
      POSTGRES_PASSWORD: Gridiron@Local123
      POSTGRES_DB: gridiron_test
    ports:
      - 5432:5432
```

---

## Known Gotchas

### EF Core Identity Map Stale Data (CRITICAL)

**Problem**: When polling EF Core for state changes made by another service/process, the identity map returns stale in-memory values even though `FirstOrDefaultAsync` executes fresh SQL.

**Root cause**: EF Core's change tracker caches entity instances. When a query returns an entity that's already tracked, EF Core returns the **tracked instance** without updating its property values from the database result.

**Symptom**: Polling loop never sees `SimulationInProgress = false` even though the background service already set it to `false` in the database.

**Fix**: Create a fresh `AsyncServiceScope` (and thus fresh `DbContext`) per poll iteration:

```csharp
// WRONG - stale identity map after first load
while (true)
{
    var league = await dbContext.Leagues.FirstOrDefaultAsync(l => l.Id == id);
    if (!league.SimulationInProgress) break;  // NEVER breaks - stale cached value
    await Task.Delay(500);
}

// CORRECT - fresh scope per poll iteration
while (true)
{
    await using var scope = serviceProvider.CreateAsyncScope();
    var freshDb = scope.ServiceProvider.GetRequiredService<GridironDbContext>();
    var league = await freshDb.Leagues.FirstOrDefaultAsync(l => l.Id == id);
    if (!league.SimulationInProgress) break;  // Sees real DB value
    await Task.Delay(500);
}
```

### Per-Test Fixture Setup Cost

**Problem**: `SimulationHttpIntegrationTests` creates a full league (4 teams x 53 players = 212+ entity inserts) in `InitializeAsync` for every single test. On a slow machine, this dominates test duration.

**Impact**: A test like `AdvanceByDays_WhenAlreadyLocked` that should be instant (check lock → return 423) took 3m 34s locally because the setup took 3+ minutes.

**CI vs Local**: CI completes 681 integration tests in ~4 minutes. A VirtualBox VM can take 15-20+ minutes for the same suite.

**Future optimization opportunity**: Share league setup across tests in the same collection instead of creating per-test. Tests that don't need full rosters could use lighter fixtures.

### Orphaned Test Databases

When test runs are killed mid-flight, uniquely-named databases (`gridiron_it_*`) accumulate in Postgres. Clean up with:

```bash
for db in $(docker exec gridiron-postgres psql -U gridironadmin -d postgres -t -c \
  "SELECT datname FROM pg_database WHERE datname LIKE 'gridiron_it_%';"); do
  docker exec gridiron-postgres psql -U gridironadmin -d postgres -c \
    "DROP DATABASE IF EXISTS \"$db\" WITH (FORCE);"
done
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
dotnet test Gridiron.IntegrationTests

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
| Repository -> DB | CRUD operations hit real Postgres | `GetByIdAsync` returns seeded entity |
| Service -> Repository | Business logic with real data access | Validation + save + retrieve |
| Controller -> Service | Full request/response cycle | HTTP 200/400/404 responses |

### Edge Cases (Include these in EVERY test class)

- [ ] Null/empty inputs -> appropriate error
- [ ] Entity not found -> 404 or null
- [ ] Duplicate creation -> constraint violation handled
- [ ] Soft delete -> excluded from normal queries
- [ ] Soft delete cascade -> related entities marked deleted
- [ ] Concurrent modification -> optimistic concurrency handled

### Error Paths (Test these, not just happy paths)

- [ ] Invalid input -> validation error returned
- [ ] Missing related entity -> appropriate error
- [ ] Database constraint violation -> handled gracefully
- [ ] Transaction rollback -> no partial state

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
- [ ] Polling loops use fresh `AsyncServiceScope` per iteration (avoid stale identity map)
