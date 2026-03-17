---
summary: "Quick connect Slack from the managed iOS app using OAuth app install or Socket Mode tokens"
read_when:
  - Implementing Slack channel setup in the iOS app
  - Understanding OAuth install and Socket Mode flows for Slack
title: "Slack — iOS Quick Connect"
sidebarTitle: "Slack"
---

# Slack — iOS Quick Connect

**Pattern: OAuth app install via `ASWebAuthenticationSession`**

Slack requires an OAuth2 "Install to workspace" flow. The app opens Slack's OAuth
authorization page (in a browser session), the user selects their workspace and approves
permissions, and Slack redirects back with an authorization code. The backend exchanges
the code for bot and app tokens, which are then written to the container.

## Option A — OAuth install (recommended for managed app)

This is the standard Slack app distribution flow. Users tap "Add to Slack" and go through
Slack's native permission screen. No tokens are copied manually.

### Pre-requisites (one-time operator setup)

1. Create a Slack app at [api.slack.com/apps](https://api.slack.com/apps).
2. Enable **Socket Mode** (Event Subscriptions → Enable Socket Mode).
3. Add the required bot scopes:
   - `channels:history`, `channels:read`
   - `chat:write`, `chat:write.customize`
   - `files:read`, `files:write`
   - `groups:history`, `groups:read`
   - `im:history`, `im:read`, `im:write`
   - `mpim:history`, `mpim:read`, `mpim:write`
   - `reactions:read`, `reactions:write`
   - `users:read`
4. Subscribe to bot events: `message.channels`, `message.groups`, `message.im`,
   `message.mpim`, `app_mention`.
5. Enable App Home → **Messages Tab**.
6. Add the redirect URL: `https://openclaw.ai/auth/callback/slack`

### User journey

```
User                iOS App              OpenClaw Backend        Slack
 │                     │                    │                      │
 │  Tap "Add Slack"    │                    │                      │
 │────────────────────►│                    │                      │
 │                     │  GET /auth/slack/start                    │
 │                     │───────────────────►│                      │
 │                     │◄───────────────────│ { authorizationURL } │
 │                     │                    │                      │
 │  ASWebAuthSession   │────────────────────────────────────────────►
 │                     │                    │                      │  User selects
 │                     │                    │                      │  workspace +
 │                     │                    │                      │  approves scopes
 │                     │◄────────────────────────────────────────────
 │                     │  Callback: openclaw.ai/auth/callback/slack?code=…
 │                     │                    │                      │
 │                     │  POST /auth/slack/exchange { code }       │
 │                     │───────────────────►│                      │
 │                     │                    │  Exchange code →     │
 │                     │                    │  bot token + app token
 │                     │                    │  Write to container  │
 │                     │◄───────────────────│                      │
 │                     │  POST /rpc/reload  │                      │
 │                     │  ✓ Connected       │                      │
 │◄────────────────────│                    │                      │
```

### iOS implementation

```swift
import AuthenticationServices

class SlackConnectViewController: UIViewController,
    ASWebAuthenticationPresentationContextProviding {

    func startOAuthFlow() async throws {
        // 1. Get the authorization URL from your backend
        let startURL = URL(string: "\(backendURL)/auth/slack/start")!
        var req = URLRequest(url: startURL)
        req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
        let (data, _) = try await URLSession.shared.data(for: req)
        let authURL = try JSONDecoder().decode(AuthStartResponse.self, from: data).url

        // 2. Open ASWebAuthenticationSession
        let session = ASWebAuthenticationSession(
            url: authURL,
            callbackURLScheme: "https"
        ) { [weak self] callbackURL, error in
            guard let url = callbackURL, error == nil else { return }
            Task { try await self?.exchangeCode(from: url) }
        }
        session.prefersEphemeralWebBrowserSession = false
        session.presentationContextProvider = self
        session.start()
    }

    func exchangeCode(from callbackURL: URL) async throws {
        guard let code = URLComponents(url: callbackURL, resolvingAgainstBaseURL: false)?
            .queryItems?.first(where: { $0.name == "code" })?.value else { return }

        var req = URLRequest(url: URL(string: "\(backendURL)/auth/slack/exchange")!)
        req.httpMethod = "POST"
        req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
        req.setValue("application/json", forHTTPHeaderField: "Content-Type")
        req.httpBody = try JSONSerialization.data(withJSONObject: ["code": code])
        let (_, resp) = try await URLSession.shared.data(for: req)
        guard (resp as? HTTPURLResponse)?.statusCode == 200 else {
            throw ConnectError.exchangeFailed
        }
        await showSuccess()
    }

    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        return view.window!
    }
}
```

### Backend token exchange

```typescript
// Pseudocode — runs on your backend
async function exchangeSlackCode(code: string, userId: string) {
  const result = await slack.oauth.v2.access({
    code,
    redirect_uri: "https://openclaw.ai/auth/callback/slack",
  });

  await fileApi.writeCredential(userId, "slack", {
    botToken: result.access_token,         // xoxb-…
    appToken: result.app_token,            // xapp-…  (Socket Mode)
    teamId: result.team.id,
    teamName: result.team.name,
  });

  await containerRPC.reloadChannel(userId, "slack");
}
```

---

## Option B — Paste tokens (Socket Mode)

For users already familiar with Slack app development, the app can accept a Bot Token
(`xoxb-…`) and App Token (`xapp-…`) directly.

### What the iOS app collects

| Field | UI label | Required | Notes |
| --- | --- | --- | --- |
| `appToken` | App token | Yes | `xapp-…` — needed for Socket Mode |
| `botToken` | Bot token | Yes | `xoxb-…` — from App Settings → Install App |

### Token validation

```swift
func validateSlackToken(_ botToken: String) async throws -> SlackTeamInfo {
    var req = URLRequest(url: URL(string: "https://slack.com/api/auth.test")!)
    req.setValue("Bearer \(botToken)", forHTTPHeaderField: "Authorization")
    let (data, _) = try await URLSession.shared.data(for: req)
    let result = try JSONDecoder().decode(SlackAuthTestResponse.self, from: data)
    guard result.ok else { throw ConnectError.invalidToken }
    return SlackTeamInfo(teamId: result.teamId, teamName: result.team)
}
```

---

## Gateway config written to container

```json5
// /data/credentials/slack.json
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: "xapp-...",
      botToken: "xoxb-...",
      dmPolicy: "pairing",
    },
  },
}
```

## Choosing between Option A and Option B

| | Option A (OAuth) | Option B (paste tokens) |
| --- | --- | --- |
| User effort | Low (click + approve) | Medium (copy from Slack portal) |
| App complexity | High | Low |
| Works for any workspace | Yes | Yes |
| Requires backend | Yes | No |
| Best for | Consumer managed app | Developers / power users |

## Related docs

- [Slack channel](/channels/slack) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/ios-quickconnect)
