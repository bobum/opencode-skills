---
name: authorization
description: JWT/Azure AD B2C authorization patterns for .NET APIs. Use when implementing authentication checks, access control, or resource-based authorization.
---

# Authorization Patterns

This skill covers JWT token handling, Azure AD integration, and access control patterns.

## Authentication Flow

```
1. User authenticates via Azure AD B2C
2. Frontend receives JWT token
3. Frontend sends token in Authorization header
4. API validates token and extracts claims
5. Controller checks authorization before data access
```

---

## Getting User Identity

### From HttpContext

```csharp
// Extension method pattern
public static class HttpContextExtensions
{
    public static string? GetAzureAdObjectId(this HttpContext context)
    {
        return context.User.FindFirst("oid")?.Value
            ?? context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    }

    public static string? GetUserEmail(this HttpContext context)
    {
        return context.User.FindFirst("emails")?.Value
            ?? context.User.FindFirst(ClaimTypes.Email)?.Value;
    }

    public static string? GetUserName(this HttpContext context)
    {
        return context.User.FindFirst("name")?.Value
            ?? context.User.FindFirst(ClaimTypes.Name)?.Value;
    }
}
```

### In Controllers

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetTeam(int id)
{
    // 1. Get user identity from JWT
    var azureAdObjectId = HttpContext.GetAzureAdObjectId();
    if (string.IsNullOrEmpty(azureAdObjectId))
    {
        return Unauthorized(new { error = "User identity not found in token" });
    }

    // 2. Check authorization before data access
    var hasAccess = await _authorizationService.CanAccessTeamAsync(azureAdObjectId, id);
    if (!hasAccess)
    {
        return Forbid();
    }

    // 3. Get and return data
    var team = await _teamRepository.GetByIdAsync(id);
    if (team == null)
    {
        return NotFound(new { error = $"Team with ID {id} not found" });
    }

    return Ok(MapToDto(team));
}
```

---

## Authorization Service Pattern

```csharp
public interface IAuthorizationService
{
    Task<bool> CanAccessLeagueAsync(string userId, int leagueId);
    Task<bool> CanAccessTeamAsync(string userId, int teamId);
    Task<bool> CanManageTeamAsync(string userId, int teamId);
    Task<bool> IsLeagueCommissionerAsync(string userId, int leagueId);
}

public class AuthorizationService : IAuthorizationService
{
    private readonly ILeagueRepository _leagueRepository;
    private readonly ITeamRepository _teamRepository;

    public async Task<bool> CanAccessTeamAsync(string userId, int teamId)
    {
        var team = await _teamRepository.GetByIdWithLeagueAsync(teamId);
        if (team == null) return false;

        // Check if user is in the league
        return await IsLeagueMemberAsync(userId, team.LeagueId);
    }

    public async Task<bool> CanManageTeamAsync(string userId, int teamId)
    {
        var team = await _teamRepository.GetByIdAsync(teamId);
        if (team == null) return false;

        // Only team owner can manage
        return team.OwnerId == userId;
    }
}
```

---

## Response Codes

| Code | When | Response |
|------|------|----------|
| **401 Unauthorized** | No token, invalid token, expired token | `Unauthorized(new { error = "..." })` |
| **403 Forbidden** | Valid token but no permission | `Forbid()` |
| **404 Not Found** | Resource doesn't exist | `NotFound(new { error = "..." })` |

### 401 vs 403

```csharp
// 401 - Identity problem
if (string.IsNullOrEmpty(azureAdObjectId))
{
    return Unauthorized(new { error = "User identity not found in token" });
}

// 403 - Permission problem (user is authenticated but can't access)
if (!await _authService.CanAccessTeamAsync(userId, teamId))
{
    return Forbid();
}
```

---

## Controller Attributes

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]  // All endpoints require authentication
public class TeamsController : ControllerBase
{
    // All actions require valid JWT

    [HttpGet]
    [ProducesResponseType(typeof(List<TeamDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> GetTeams()
    {
        // ...
    }

    [HttpPost]
    [ProducesResponseType(typeof(TeamDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    [ProducesResponseType(StatusCodes.Status403Forbidden)]
    public async Task<IActionResult> CreateTeam(CreateTeamRequest request)
    {
        // ...
    }
}
```

---

## Resource-Based Authorization

For actions on specific resources:

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> UpdateTeam(int id, UpdateTeamRequest request)
{
    var userId = HttpContext.GetAzureAdObjectId();
    if (string.IsNullOrEmpty(userId))
    {
        return Unauthorized(new { error = "User identity not found" });
    }

    // Check team ownership
    if (!await _authService.CanManageTeamAsync(userId, id))
    {
        _logger.LogWarning("User {UserId} attempted to update team {TeamId} without permission",
            userId, id);
        return Forbid();
    }

    // Proceed with update
    var result = await _teamService.UpdateAsync(id, request);
    return Ok(result);
}
```

---

## League-Level Access

Common pattern for league membership:

```csharp
public async Task<IActionResult> GetLeagueData(int leagueId)
{
    var userId = HttpContext.GetAzureAdObjectId();
    if (string.IsNullOrEmpty(userId))
    {
        return Unauthorized(new { error = "User identity not found" });
    }

    // Check league membership
    var membership = await _leagueRepository.GetMembershipAsync(leagueId, userId);
    if (membership == null)
    {
        return Forbid();
    }

    // User is in league - return data
    // Optionally check membership.Role for commissioner-only actions
}
```

---

## Testing Authorization

### Controller Tests

```csharp
[Fact]
public async Task GetTeam_WhenNotInLeague_Returns403()
{
    // Arrange
    var controller = new TeamsController(_mockService.Object, _mockAuth.Object);
    _mockAuth.Setup(a => a.CanAccessTeamAsync(It.IsAny<string>(), 1))
        .ReturnsAsync(false);

    // Act
    var result = await controller.GetTeam(1);

    // Assert
    Assert.IsType<ForbidResult>(result);
}

[Fact]
public async Task GetTeam_WhenNoToken_Returns401()
{
    // Arrange - HttpContext with no user claims
    var controller = new TeamsController(_mockService.Object, _mockAuth.Object);
    controller.ControllerContext = new ControllerContext
    {
        HttpContext = new DefaultHttpContext()  // No claims
    };

    // Act
    var result = await controller.GetTeam(1);

    // Assert
    var unauthorized = Assert.IsType<UnauthorizedObjectResult>(result);
    Assert.Contains("identity", unauthorized.Value?.ToString());
}
```

---

## Security Requirements

> **Think like a security engineer WHILE writing auth code.**

### OWASP Top 10 (Verify for every endpoint)

| Vulnerability | Check |
|---------------|-------|
| **Injection** | User input parameterized, not concatenated |
| **Broken Auth** | Token validated, expiration checked |
| **Sensitive Data** | No PII in logs, errors don't leak info |
| **Access Control** | Authorization checked BEFORE data access |
| **IDOR** | Can't access other users' resources by changing ID |

### Authorization Flow (Every endpoint must follow this)

```csharp
// 1. FIRST - Get identity
var userId = HttpContext.GetAzureAdObjectId();
if (string.IsNullOrEmpty(userId))
    return Unauthorized();  // 401 - no identity

// 2. SECOND - Check permission BEFORE touching data
if (!await _authService.CanAccessResource(userId, resourceId))
    return Forbid();  // 403 - no permission

// 3. ONLY THEN - Access data
var resource = await _repository.GetByIdAsync(resourceId);
```

### Common Vulnerabilities (Avoid these)

| Vulnerability | Example | Fix |
|---------------|---------|-----|
| Mass assignment | `_mapper.Map(request, entity)` overwrites protected fields | Use explicit property assignment or `[Bind]` |
| IDOR | `GET /api/users/123` returns any user | Check `userId == requestingUserId` |
| Privilege escalation | User can set `IsAdmin = true` | Don't accept role changes from user input |
| Info disclosure | Error shows stack trace | Return generic errors, log details server-side |

### Logging (Include for every auth decision)

```csharp
// Log failures with context (but not tokens!)
_logger.LogWarning("User {UserId} denied access to team {TeamId}", userId, teamId);

// Don't log sensitive data
_logger.LogWarning("Token: {Token}", token);  // NEVER DO THIS
```

### Required Test Coverage (Write these as you implement)

| Scenario | Expected | Test |
|----------|----------|------|
| No token | 401 | Request without Authorization header |
| Invalid token | 401 | Request with malformed token |
| Valid token, no permission | 403 | User A accessing User B's resource |
| Valid token, has permission | 200 | User accessing own resource |
| Cross-user access | 403 | Explicitly try to access another user's data |

---

## Checklist for New Endpoint

- [ ] `[Authorize]` attribute present
- [ ] User ID extracted from `HttpContext.GetAzureAdObjectId()`
- [ ] Returns 401 if no user identity
- [ ] Authorization checked BEFORE data access
- [ ] Returns 403 if not authorized (not 404 - don't hide existence)
- [ ] No IDOR vulnerability (can't access others' data by changing ID)
- [ ] No mass assignment (explicit property mapping)
- [ ] Unauthorized attempts logged (without sensitive data)
- [ ] Error messages don't leak internal details
- [ ] Tests cover: no token, invalid token, wrong user, correct user
