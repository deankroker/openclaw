---
summary: "Quick connect Microsoft Teams from the managed iOS app using Azure AD OAuth"
read_when:
  - Implementing Microsoft Teams channel setup in the iOS app
  - Understanding Azure AD OAuth and Bot Framework flows for Teams
title: "Microsoft Teams — iOS Quick Connect"
sidebarTitle: "Microsoft Teams"
---

# Microsoft Teams — iOS Quick Connect

**Pattern: Azure AD OAuth (enterprise SSO)**

Microsoft Teams uses the Azure Bot Framework. Connecting requires an Azure AD application
registration, a Bot Framework bot, and tenant admin consent for the required permissions.
This is the most complex channel to set up, but it also provides the most seamless enterprise
SSO experience — users authenticate with their existing Microsoft 365 identity.

<Note>
Teams bots require **tenant admin consent** for the required permissions. In a managed app,
you will typically pre-register a multi-tenant Azure AD app and ask the user's IT admin to
consent once. Individual users can then connect their Teams account without IT involvement
after that initial consent.
</Note>

## Architecture

```
iOS App  ──►  Azure AD OAuth  ──►  OpenClaw Backend
                                        │
                              Bot Framework Registration
                                        │
                              OpenClaw Container (Gateway)
                                        │
                              Teams Bot Framework API
```

## Pre-requisites (one-time operator setup)

1. Create an Azure AD app registration (multi-tenant or single-tenant).
2. Add **delegated permissions**: `Chat.ReadWrite`, `ChannelMessage.Send`, `User.Read`.
3. Add a redirect URI: `https://openclaw.ai/auth/callback/msteams`
4. Create a Bot Framework bot at [dev.botframework.com](https://dev.botframework.com) or via
   Azure Bot Service.
5. Note the **Microsoft App ID** and **App Password** (client secret).
6. Add the Teams channel to the bot in Azure Bot Service.

## Option A — Azure AD OAuth (delegated, user identity)

The user signs in with their Microsoft 365 account. The app gets an access token scoped to
the user's Teams tenant and writes it to the container.

### User journey

```
User                iOS App              OpenClaw Backend        Azure AD / Teams
 │                     │                    │                          │
 │  Tap "Add Teams"    │                    │                          │
 │────────────────────►│                    │                          │
 │                     │  GET /auth/msteams/start                     │
 │                     │───────────────────►│                          │
 │                     │◄───────────────────│  { authorizationURL }    │
 │                     │                    │                          │
 │  ASWebAuthSession   │──────────────────────────────────────────────►
 │                     │                    │                          │  User logs in
 │                     │                    │                          │  with Microsoft
 │                     │                    │                          │  account + consent
 │                     │◄──────────────────────────────────────────────
 │                     │  Callback: openclaw.ai/auth/callback/msteams?code=…
 │                     │                    │                          │
 │                     │  POST /auth/msteams/exchange { code }        │
 │                     │───────────────────►│                          │
 │                     │                    │  Exchange → tokens       │
 │                     │                    │  Write to container      │
 │                     │◄───────────────────│                          │
 │                     │  ✓ Connected       │                          │
 │◄────────────────────│                    │                          │
```

### iOS implementation

```swift
import AuthenticationServices

class TeamsConnectViewController: UIViewController,
    ASWebAuthenticationPresentationContextProviding {

    let tenantId = "common"  // or a specific tenant ID for single-tenant apps

    func startOAuthFlow() async throws {
        // 1. Build Microsoft OAuth2 authorization URL
        var components = URLComponents(string:
            "https://login.microsoftonline.com/\(tenantId)/oauth2/v2.0/authorize")!
        components.queryItems = [
            URLQueryItem(name: "client_id", value: msClientID),
            URLQueryItem(name: "response_type", value: "code"),
            URLQueryItem(name: "redirect_uri", value: "https://openclaw.ai/auth/callback/msteams"),
            URLQueryItem(name: "scope", value: "Chat.ReadWrite ChannelMessage.Send User.Read offline_access"),
            URLQueryItem(name: "state", value: generateCSRFState()),
            URLQueryItem(name: "prompt", value: "select_account"),
        ]

        // 2. Open with ASWebAuthenticationSession
        let session = ASWebAuthenticationSession(
            url: components.url!,
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

        var req = URLRequest(url: URL(string: "\(backendURL)/auth/msteams/exchange")!)
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
async function exchangeTeamsCode(code: string, userId: string) {
  const tokens = await msal.acquireTokenByCode({ code, scopes: [...] });

  await fileApi.writeCredential(userId, "msteams", {
    tenantId: tokens.tenantId,
    accessToken: tokens.accessToken,
    refreshToken: tokens.refreshToken,
    // Bot Framework credentials (static, shared across all users)
    msAppId: process.env.MS_APP_ID,
    msAppPassword: process.env.MS_APP_PASSWORD,
  });

  await containerRPC.reloadChannel(userId, "msteams");
}
```

---

## Option B — Bot-only connection (application identity)

If your use case is a Teams bot that responds in channels (not on behalf of a user), use
application-level credentials instead of delegated OAuth. This requires an Azure AD app with
**application permissions** (`Chat.ReadWrite.All`, `ChannelMessage.Send` — requires admin
consent globally).

The iOS app collects only the **tenant ID** from the user (their organization's domain or
tenant GUID), and your backend handles the rest with the pre-configured bot credentials:

```swift
struct TeamsAppCredential: Encodable {
    let tenantId: String    // User's tenant ID (e.g., "contoso.onmicrosoft.com")
}
```

---

## Admin consent

For multi-tenant apps, direct the tenant admin to:

```
https://login.microsoftonline.com/{tenantId}/adminconsent
  ?client_id={your-client-id}
  &redirect_uri=https://openclaw.ai/auth/callback/msteams/admin-consent
```

Include an in-app "Contact IT Admin" button that generates this URL and copies it to the
clipboard or shares it via the iOS share sheet.

---

## Gateway config written to container

```json5
// /data/credentials/msteams.json
{
  channels: {
    msteams: {
      enabled: true,
      msAppId: "aaaabbbb-cccc-dddd-eeee-ffffffffffff",
      msAppPassword: "app-secret",
      tenantId: "contoso.onmicrosoft.com",
      dmPolicy: "pairing",
    },
  },
}
```

## Limitations

- **Admin consent required**: The tenant administrator must approve the app before any user
  in that organization can connect.
- **Enterprise only**: Teams is primarily used in corporate environments. Consumer Microsoft
  accounts may have limited Teams access.
- **Token refresh**: Microsoft access tokens expire every 1 hour. The container Gateway must
  use the refresh token to obtain new access tokens automatically.

## Related docs

- [Microsoft Teams channel](/channels/msteams) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/ios-quickconnect)
