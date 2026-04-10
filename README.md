# 🔐 JwtServicePackage

A **production-ready JWT authentication service** for .NET, designed with **security, scalability, and real-world constraints** in mind.

![NuGet Version](https://img.shields.io/nuget/v/JwtServicePackage)
![NuGet Downloads](https://img.shields.io/nuget/dt/JwtServicePackage)

Unlike basic JWT implementations, this package provides a **full token lifecycle system**, including:

* Access & Refresh tokens
* Token rotation
* Token revocation (blacklisting)
* Replay attack detection
* Key rotation
* Multi-device session control

---

# 🚀 Features

## ✅ Core Authentication

* Generate **Access + Refresh token pairs**
* Validate tokens with custom logic
* Decode tokens safely

## 🔁 Token Lifecycle Management

* Refresh token rotation (secure renewal)
* Revoke individual tokens
* Revoke all tokens per user
* Track active sessions

## 🔐 Security Enhancements

* Token replay attack detection (JTI tracking)
* Token blacklisting
* Hash-based refresh token storage
* Claims-based identity

## ⚙️ Operational Features

* Background cleanup of expired tokens
* Configurable limits per user

---

# 📦 Installation

Add the package to your solution (local or NuGet):

```bash
dotnet add package JwtServicePackage
```

---

# ⚙️ Configuration

Add to `appsettings.json`:

```json
{
  "JwtSettings": {
    "SecretKey": "my-super-secret-key-very-long-32+chars",
    "Issuer": "your-app",
    "Audience": "your-app-users",
    "AccessTokenExpiryMinutes": 15,
    "RefreshTokenExpiryDays": 7,
    "EnableKeyRotation": true,
    "KeyRotationIntervalDays": 7,
    "EnableTokenReplayDetection": true,
    "EnableTokenBlacklisting": true,
    "MaxActiveTokensPerUser": 5
  }
}
```

---

# 🧩 Setup (Program.cs)

```csharp
builder.Services.AddJwtAuthentication(builder.Configuration);
builder.Services.AddHttpContextAccessor();
```

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

---

# 🔑 Generating Tokens

```csharp
var tokens = _jwtService.GenerateTokenPair(
    userId: user.UserId.ToString(),
    customClaims: new Dictionary<string, object>
    {
        { ClaimTypes.Email, user.Email }
    },
    deviceInfo: "web",
    ipAddress: "127.0.0.1"
);
```

---

# 🔍 Validating Tokens

```csharp
var result = _jwtService.ValidateAccessToken(token);

if (!result.IsValid)
{
    // handle invalid token
}
```

---

# 🔄 Refreshing Tokens

```csharp
var newTokens = _jwtService.RefreshToken(refreshToken);
```

---

# 🚫 Revoking Tokens

### Revoke Access Token

```csharp
_jwtService.RevokeToken(accessToken);
```

### Revoke Refresh Token

```csharp
_jwtService.RevokeRefreshToken(refreshToken);
```

### Revoke All User Sessions

```csharp
_jwtService.RevokeAllUserTokens(userId);
```

---

# 👤 Access Current User

Use claims via `HttpContext`:

```csharp
var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
```

---

# 🧠 Design Philosophy

## Why not just use default JWT middleware?

Most JWT examples:

* Use a single static secret ❌
* Cannot revoke tokens ❌
* Cannot detect replay attacks ❌
* Break under scaling ❌

This package solves those problems by making:

> 🧠 **The JWT Service the source of truth**, not middleware

### Middleware becomes:

* A gatekeeper

### JwtService becomes:

* The authority for validation, rotation, and security

---

# 🏗 Architecture Overview

```
Client
   ↓
JWT Middleware (basic validation)
   ↓
JwtService (real validation)
   ↓
Controllers / Services
```

---

# 🧪 Testing

You can test quickly:

```csharp
var tokens = _jwtService.GenerateTokenPair("user-123");
Console.WriteLine(tokens.AccessToken);
```

Use with:

```http
Authorization: Bearer <token>
```

---

# 🔄 Background Cleanup

Token cleanup runs automatically via `BackgroundService`:

* Removes expired refresh tokens
* Clears old revoked tokens
* Cleans replay tracking

---

# 📄 License

MIT License - free for commercial and personal use.

---

# Author

Created and Maintained by: [Ethern-Myth](https://github.com/ethern-myth)
