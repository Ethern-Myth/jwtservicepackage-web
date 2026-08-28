# 🔐 JwtServicePackage

A **production-ready JWT authentication and token lifecycle engine** for .NET, designed for security, scalability, and real-world distributed systems.

![NuGet Version](https://img.shields.io/nuget/v/JwtServicePackage)
![NuGet Downloads](https://img.shields.io/nuget/dt/JwtServicePackage)

Unlike traditional JWT setups that rely on static secrets and stateless validation only, this package introduces a full token security lifecycle system with:

- Key rotation
- Multi-key validation (zero-downtime rotation)
- Refresh token lifecycle management
- Token revocation (blacklisting)
- Replay attack detection
- Session control per user/device

---

## Updates

### Lastest updates

- Updated EnableTokenReplayDetection to false as default
- Added EnableTokenReplayDetectionMinutes for user configuration by default set to 5 minutes
- EnableTokenReplayDetectionMinutes can be added to `JwtSettings` if not, by default will be false
 
**New version**: **10.0.10** is available with updates to address the above changes

**IMPORTANT**: Version 10.0.10 is the latest, 10.0.5 is available alternatively

## 🚀 Features

### 🔐 Authentication Core

- Generate **Access + Refresh token pairs**
- Claims-based identity support
- Token decoding utilities

### 🔁 Token Lifecycle Management

- Secure refresh token rotation
- Per-user session limits
- Token revocation (single or all sessions)
- Device-aware session tracking

### 🧠 Security Enhancements

- Replay attack detection (JTI tracking)
- Token blacklisting
- Hash-based refresh token storage
- Per-user active session enforcement

### 🔄 Key Management System (NEW)

- Automatic key generation (if not provided)
- Rotating signing keys with KeyId (kid)
- Multi-key validation for backward compatibility
- Retains old keys until refresh-token expiry window ends
- Zero-downtime key rotation

---

### 🧠 Architecture Overview

Client → JWT Middleware → JwtService → Controller

Key rotation ensures all valid keys remain usable during rotation windows.

---

## 📦 Installation

```bash
dotnet add package JwtServicePackage
```

---

## ⚙️ Configuration

Add to `appsettings.json`:

```json
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

---

## 🧩 Setup (Program.cs)

```csharp
builder.Services.AddJwtAuthentication(builder.Configuration);
builder.Services.AddHttpContextAccessor();
```

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

---

## 🔑 Usage

Generate tokens:

```csharp

var tokens = _jwtService.GenerateTokenPair("user-123");

Validate:
var result = _jwtService.ValidateAccessToken(token);

Refresh:
var newTokens = _jwtService.RefreshToken(refreshToken);
```

## 🔑 Generating Tokens

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

## 🔍 Validating Tokens

```csharp
var result = _jwtService.ValidateAccessToken(token);

if (!result.IsValid)
{
    // handle invalid token
}
```

## 🔄 Refreshing Tokens

```csharp
var newTokens = _jwtService.RefreshToken(refreshToken);
```

---

## 🚫 Revocation

```csharp
_jwtService.RevokeToken(accessToken);
_jwtService.RevokeRefreshToken(refreshToken);
_jwtService.RevokeAllUserTokens(userId);
```

---

## 👤 Access Current User

Use claims via `HttpContext`:

```csharp
var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
```

---

## 🔐 Security Model

JWT Middleware → cryptographic validation  
JwtService → business security rules

Do NOT duplicate validation logic.

---

## 🔄 Background Cleanup

Token cleanup runs automatically via `BackgroundService`:

* Removes expired refresh tokens
* Clears old revoked tokens
* Cleans replay tracking

---

## 📄 License

MIT License - free for commercial and personal use.

---

## 👨‍💻 Author

Created and Maintained by: [Ethern-Myth](https://github.com/ethern-myth)
