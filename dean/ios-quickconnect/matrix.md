---
summary: "Quick connect Matrix from the managed iOS app using an access token or password login"
read_when:
  - Implementing Matrix channel setup in the iOS app
  - Understanding access token and SSO flows for Matrix
title: "Matrix — iOS Quick Connect"
sidebarTitle: "Matrix"
---

# Matrix — iOS Quick Connect

**Pattern: access token (simple) or username + password login (full e2e)**

Matrix is a federated protocol. Users can connect to any Matrix homeserver (matrix.org,
Element Matrix Services, or self-hosted). The iOS app collects the homeserver URL and either
a pre-generated access token or the user's Matrix credentials to obtain one.

## Option A — Paste access token (simple)

Best for users who already have a Matrix account and know how to generate a token. No OAuth
redirect needed.

### What the iOS app collects

| Field | UI label | Required | Notes |
| --- | --- | --- | --- |
| `homeserverUrl` | Homeserver URL | Yes | Example: `https://matrix.org` |
| `accessToken` | Access token | Yes | From Element → Settings → Help & About → Access Token |
| `userId` | Matrix user ID | Yes | Example: `@username:matrix.org` |
| `deviceId` | Device ID | No | Auto-detected from token if omitted |

### Token validation

```swift
struct MatrixCredential: Codable {
    let homeserverUrl: String
    let accessToken: String
    let userId: String
    let deviceId: String?
}

func validateMatrixToken(_ cred: MatrixCredential) async throws -> MatrixWhoAmI {
    // Matrix Client-Server API: GET /_matrix/client/v3/account/whoami
    var req = URLRequest(url: URL(string: "\(cred.homeserverUrl)/_matrix/client/v3/account/whoami")!)
    req.setValue("Bearer \(cred.accessToken)", forHTTPHeaderField: "Authorization")
    let (data, response) = try await URLSession.shared.data(for: req)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw ConnectError.invalidToken
    }
    return try JSONDecoder().decode(MatrixWhoAmI.self, from: data)
}
// MatrixWhoAmI: { user_id: String, device_id: String }
```

---

## Option B — Username + password login

The app logs in to the Matrix homeserver on behalf of the user, obtains an access token, and
stores it in the container. This keeps the raw token out of user hands entirely.

### What the iOS app collects

| Field | UI label | Required | Notes |
| --- | --- | --- | --- |
| `homeserverUrl` | Homeserver URL | Yes | Example: `https://matrix.org` |
| `username` | Matrix username | Yes | Without `@server` suffix |
| `password` | Password | Yes | Used once to get a token; not stored |

### Login flow

```swift
struct MatrixLoginRequest: Encodable {
    let type = "m.login.password"
    let identifier: MatrixIdentifier
    let password: String
    let initial_device_display_name = "OpenClaw"

    struct MatrixIdentifier: Encodable {
        let type = "m.id.user"
        let user: String
    }
}

struct MatrixLoginResponse: Decodable {
    let access_token: String
    let device_id: String
    let user_id: String
}

func loginToMatrix(homeserver: String, username: String, password: String) async throws -> MatrixLoginResponse {
    let url = URL(string: "\(homeserver)/_matrix/client/v3/login")!
    var req = URLRequest(url: url)
    req.httpMethod = "POST"
    req.setValue("application/json", forHTTPHeaderField: "Content-Type")
    req.httpBody = try JSONEncoder().encode(MatrixLoginRequest(
        identifier: .init(user: username),
        password: password
    ))
    let (data, response) = try await URLSession.shared.data(for: req)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw ConnectError.loginFailed
    }
    return try JSONDecoder().decode(MatrixLoginResponse.self, from: data)
}
```

After a successful login, store only the `access_token` and `device_id` — never persist the
password beyond the login request.

---

## Option C — SSO (Element/MSC3824)

Some Matrix homeservers support SSO via OIDC. The iOS app can open
`<homeserver>/_matrix/client/v3/login/sso/redirect?redirectUrl=<callback>` in an
`ASWebAuthenticationSession`. The homeserver redirects back with a login token that can be
exchanged for an access token:

```
GET /_matrix/client/v3/login/sso/redirect?redirectUrl=https://openclaw.ai/auth/callback/matrix
→ User authenticates with SSO provider
→ Redirected to https://openclaw.ai/auth/callback/matrix?loginToken=<token>

POST /_matrix/client/v3/login
{
  "type": "m.login.token",
  "token": "<loginToken>"
}
→ { access_token, device_id, user_id }
```

<Note>
SSO support varies by homeserver. matrix.org and Element Matrix Services support OIDC-based
SSO. Self-hosted Synapse/Dendrite may or may not have it configured.
</Note>

---

## Homeserver discovery

Matrix uses `.well-known/matrix/client` for server discovery. Before showing any form fields,
resolve the homeserver from the user's Matrix ID (if they enter `@user:matrix.org`):

```swift
func resolveHomeserver(from matrixId: String) async throws -> String {
    // Extract domain from @user:domain — handle IDs with multiple colons (e.g. @user:server:port)
    // Drop the leading "@user:" prefix and treat the rest as the domain
    guard matrixId.hasPrefix("@"),
          let colonRange = matrixId.range(of: ":") else {
        throw ConnectError.invalidId
    }
    let domain = String(matrixId[matrixId.index(after: colonRange.lowerBound)...])
    let wellKnownURL = URL(string: "https://\(domain)/.well-known/matrix/client")!
    if let (data, response) = try? await URLSession.shared.data(from: wellKnownURL),
       (response as? HTTPURLResponse)?.statusCode == 200,
       let json = try? JSONDecoder().decode(MatrixWellKnown.self, from: data) {
        return json.homeserver.base_url
    }
    // Fallback: use the domain directly
    return "https://\(domain)"
}
```

---

## Gateway config written to container

```json5
// /data/credentials/matrix.json
{
  channels: {
    matrix: {
      enabled: true,
      homeserverUrl: "https://matrix.org",
      accessToken: "syt_...",
      userId: "@yourbotname:matrix.org",
      deviceId: "OPENCLAW_BOT",
      dmPolicy: "pairing",
      allowFrom: ["@youruser:matrix.org"],
    },
  },
}
```

## Choosing between options

| | Option A (paste token) | Option B (password login) | Option C (SSO) |
| --- | --- | --- | --- |
| User effort | Medium (copy from Element) | Low (type username/password) | Low (click + approve) |
| Password seen by app | No | Yes (used once only) | No |
| Works on all homeservers | Yes | Yes | Partial |
| Best for | Power users | Consumer managed app | Enterprise homeservers |

## Related docs

- [Matrix channel](/channels/matrix) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/dean/ios-quickconnect)
