# 🔐 JwtServicePackage

A **production-ready JWT authentication and token lifecycle engine** for .NET, designed for security, scalability, and real-world distributed systems.

![NuGet Version](https://img.shields.io/nuget/v/JwtServicePackage)
![NuGet Downloads](https://img.shields.io/nuget/dt/JwtServicePackage)

Unlike traditional JWT setups that rely on static secrets and stateless validation only, this package introduces a full token security lifecycle system with:

- Key rotation
- Multi-key validation (zero-downtime rotation)
- Refresh token lifecycle management
- Automatic claims preservation on refresh
- Token revocation (blacklisting)
- Replay attack detection
- Session control per user/device

---

## 🚀 Updates & Release Notes

### 🌟 What's New in Version 10.0.20

- **Automatic Claims Preservation on Refresh**: `RefreshToken()` can now take an optional `expiredAccessToken`. If no new claims are provided, the service extracts and carries forward the claims from the previous token automatically—eliminating redundant database lookups during token renewal.
- **Flexible Claims Overloads**: Support for `IEnumerable<Claim>` alongside `Dictionary<string, object>` across token generation and refresh methods.
- **Enhanced Token & Principal Helpers**: Added `GetPrincipalFromExpiredToken()`, `GetUserIdFromPrincipal()`, `GetClaimFromToken<T>()`, and generic claims access via `TokenValidationResult.GetClaimValue<T>()`.
- **Absolute Refresh Expiration Window**: Refresh token rotation retains the original absolute session expiration window configured in `appsettings.json` (`RefreshTokenExpiryDays`).
- **DI & Extension Method Cleanups**: Fixed static DI extension method signatures (`this IServiceCollection`) and key resolver performance in `AddJwtAuthentication`.

> **IMPORTANT**: Version **10.0.20** is the latest recommended release. Users on `10.0.10` or earlier are encouraged to upgrade.

---

## 🚀 Features

### 🔐 Authentication Core

- Generate **Access + Refresh token pairs** using standard `Claim` collections or dictionary key-value maps.
- Claims-based identity support with built-in extraction helpers.
- Fast token decoding and `ClaimsPrincipal` extraction utilities.

### 🔁 Token Lifecycle Management

- Smart refresh token rotation with claim preservation.
- Absolute persistence window matching `appsettings.json`.
- Per-user active session limits (`MaxActiveTokensPerUser`).
- Revocation options (single access token, refresh token, or all user sessions).
- Device and IP-aware session tracking.

### 🧠 Security Enhancements

- Replay attack detection (JTI tracking) with configurable window (`EnableTokenReplayDetectionMinutes`).
- Token blacklisting for instant access revocation.
- Cryptographic hash-based refresh token storage.
- Active per-user session limits.

### 🔄 Key Management System

- Automatic key generation if not provided.
- Rotating signing keys with KeyId (`kid`).
- Multi-key validation for zero-downtime key rotation.
- Old signing keys retained during the active refresh window.

---

## 📦 Installation

```bash
dotnet add package JwtServicePackage --version 10.0.20
```

## ⚙️ Configuration

Add to appsettings.json:

```JSON
{
  "JwtSettings": {
    "SecretKey": "your-initial-secret-key-32chars-minimum",
    "Issuer": "your-app",
    "Audience": "your-app-users",
    "AccessTokenExpiryMinutes": 15,
    "RefreshTokenExpiryDays": 7,
    "EnableKeyRotation": true,
    "KeyRotationIntervalDays": 7,
    "EnableTokenReplayDetection": false,
    "EnableTokenReplayDetectionMinutes": 5,
    "EnableTokenBlacklisting": true,
    "MaxActiveTokensPerUser": 5
  }
}
```

## 🧩 Setup (Program.cs)

```csharp
builder.Services.AddJwtAuthentication(builder.Configuration);
builder.Services.AddHttpContextAccessor();

app.UseAuthentication();
app.UseAuthorization();
```

## 🔑 Usage

1. Generating Tokens

Using standard Claim collections (Recommended):

```csharp
var claims = new List<Claim>
{
    new(ClaimTypes.Email, user.Email),
    new(ClaimTypes.Name, $"{user.FirstName} {user.LastName}"),
    new(ClaimTypes.Role, user.RoleKey)
};

// Add multi-value claims such as permissions easily
foreach (var perm in userPermissions)
{
    claims.Add(new Claim("permissions", perm));
}

var tokens = _jwtService.GenerateTokenPair(
    userId: user.UserId.ToString(),
    claims: claims,
    deviceInfo: Request.Headers.UserAgent.ToString(),
    ipAddress: HttpContext.Connection.RemoteIpAddress?.ToString()
);
```

Or using Dictionary<string, object>:

```csharp
var customClaims = new Dictionary<string, object>
{
    [ClaimTypes.Email] = user.Email,
    [ClaimTypes.Role] = user.RoleKey,
    ["permissions"] = new[] { "read:users", "write:users" }
    //OR
    ["permissions"] = string.Join(",", new[] { "read:users", "write:users" })
};

var tokens = _jwtService.GenerateTokenPair(
    userId: user.UserId.ToString(),
    customClaims: customClaims
);
```

2. Validating Tokens

```csharp
var result = _jwtService.ValidateAccessToken(accessToken);

if (!result.IsValid)
{
    // Handle invalid/expired token
    var error = result.ErrorMessage;
    var errorType = result.ErrorType; // e.g. TokenValidationErrorType.Expired
}

// Extract strongly-typed claims directly from validation result
var email = result.GetClaimValue<string>(ClaimTypes.Email);
```

3. Smart Token Refresh (Zero Database Overhead)

In v10.0.20, passing the expiredAccessToken automatically preserves all claims from the expired token into the new access token—saving unnecessary database calls:

```csharp
[HttpPost("refresh")]
public IActionResult Refresh([FromBody] RefreshRequestDto request)
{
    // Refreshes the pair, retains existing claims, and maintains original session expiry window
    var tokenPair = _jwtService.RefreshToken(
        refreshToken: request.RefreshToken,
        expiredAccessToken: request.AccessToken,
        updatedClaims: null, // Pass null to automatically preserve claims from expired token
        newDeviceInfo: Request.Headers.UserAgent.ToString(),
        newIpAddress: HttpContext.Connection.RemoteIpAddress?.ToString()
    );

    return Ok(tokenPair);
}
```

*Note*: If user roles or permissions have changed, you can pass updated claims into updatedClaims to override the old payload.

4. Extraction & Helper Utilities

```csharp
// Get ClaimsPrincipal from an expired access token (lifetime check ignored)
ClaimsPrincipal? principal = _jwtService.GetPrincipalFromExpiredToken(expiredAccessToken);

// Extract User ID directly from a principal
string? userId = _jwtService.GetUserIdFromPrincipal(principal);

// Extract a specific claim value directly from a token string
string? userEmail = _jwtService.GetClaimFromToken<string>(token, ClaimTypes.Email);

// Decode raw token claims map
Dictionary<string, object> claimsMap = _jwtService.DecodeToken(token);
```

## 🚫 Revocation

```csharp
// Revoke a specific access token JTI
_jwtService.RevokeToken(accessToken, reason: "User logged out");

// Revoke a refresh token
_jwtService.RevokeRefreshToken(refreshToken, reason: "Security rotation");

// Revoke all active sessions for a user (e.g., password reset)
_jwtService.RevokeAllUserTokens(userId, reason: "Password changed");
```

## 👤 Access Current User in Controllers

Use standard ASP.NET Core HttpContext claims:

```csharp
var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
var userEmail = HttpContext.User.FindFirst(ClaimTypes.Email)?.Value;
```

## 🔐 Security Model

```text
Client Request ──► JwtBearer Middleware ──► Cryptographic Validation
                        │
                        ▼
                  JwtService ──► Business Security Rules (Blacklist / Replay / Revocation)
```

- JWT Middleware: Handles signature, expiration, issuer, and audience validation.

- JwtService: Handles lifecycle state, blacklisting, replay protection, and key rotation management.

## 🔄 Background Cleanup

Automatic background cleanup runs hourly via BackgroundService:

- Purges expired and revoked refresh tokens.

- Cleans old revoked JTIs older than 7 days.

- Flushes expired JTI entries from the replay tracking cache.

## 📄 License

MIT License - free for commercial and personal use.

## 👨‍💻 Author

Created and Maintained by: [Ethern-Myth](https://github.com/ethern-myth)