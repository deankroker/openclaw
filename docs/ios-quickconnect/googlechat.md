---
summary: "Quick connect Google Chat from the managed iOS app using a service account or Google OAuth"
read_when:
  - Implementing Google Chat channel setup in the iOS app
  - Understanding service account and Google OAuth flows for Google Chat
title: "Google Chat — iOS Quick Connect"
sidebarTitle: "Google Chat"
---

# Google Chat — iOS Quick Connect

**Pattern: service account JSON (operator-provisioned) or Google OAuth (user identity)**

Google Chat bots require a Google Workspace account and a Google Cloud project. There are two
patterns: a **service account** (operator-managed, best for fully managed apps) or **Google
OAuth** (user-managed, best when users want to connect their own Workspace account).

<Note>
Google Chat bots are only available to Google Workspace accounts. Free Gmail accounts
(`@gmail.com`) cannot use the Google Chat API as an app destination.
</Note>

## Option A — Service account (operator-provisioned, recommended)

The operator creates one Google Cloud project and service account. Each user connects to the
same bot without needing to touch the Google Cloud Console. This is the smoothest managed-app
experience.

### Pre-requisites (one-time operator setup)

1. Create a Google Cloud project.
2. Enable the **Google Chat API**.
3. Create a **Service Account** with the `Chat Bot` role (or custom role with `chat.messages.*` permissions).
4. Download the service account JSON key file.
5. In the Google Chat API configuration, publish the bot (App Name, Avatar, Description,
   Functionality: Direct Messages + Spaces).
6. Set up the connection type: **App URL** pointing to your backend webhook endpoint, or
   **Cloud Pub/Sub** for push delivery.

### User journey (service account)

```
User                iOS App              OpenClaw Backend
 │                     │                    │
 │  Tap "Add Google    │                    │
 │  Chat"              │                    │
 │────────────────────►│                    │
 │                     │  POST /auth/googlechat/provision             │
 │                     │  { supabaseJWT }                             │
 │                     │───────────────────►│                         │
 │                     │                    │  Write shared service   │
 │                     │                    │  account cred to        │
 │                     │                    │  user's container       │
 │                     │◄───────────────────│                         │
 │                     │  Show "Add to      │                         │
 │                     │  Spaces" button    │                         │
 │◄────────────────────│                    │                         │
 │  Tap to add bot to  │                    │                         │
 │  a Google Space     │                    │                         │
 │────────────────────►│                    │                         │
 │                     │  Opens Google      │                         │
 │                     │  Chat deep link    │                         │
 │                     │  ✓ Connected       │                         │
 │◄────────────────────│                    │                         │
```

The iOS app shows a button that deep-links the user into Google Chat to add the bot:

```swift
func addBotToGoogleChat() {
    // Deep link to add the bot to Google Chat
    let botName = "openclaw"  // Bot's short name in Workspace
    if let url = URL(string: "https://chat.google.com/dm/\(botName)") {
        UIApplication.shared.open(url)
    }
}
```

---

## Option B — Google OAuth (user identity)

The user grants the app permission to act on their behalf using Google OAuth2. This is more
complex but allows users to connect their own Google Workspace account.

### Pre-requisites (one-time operator setup)

1. Create a Google Cloud project with the **Google Chat API** enabled.
2. Create an **OAuth 2.0 Client ID** (type: Web application).
3. Add the redirect URI: `https://openclaw.ai/auth/callback/googlechat`
4. Note the **Client ID** and **Client Secret** (kept server-side).

### User journey

```
User                iOS App              OpenClaw Backend        Google
 │                     │                    │                      │
 │  Tap "Add Google    │                    │                      │
 │  Chat"              │                    │                      │
 │────────────────────►│                    │                      │
 │                     │  GET /auth/googlechat/start                │
 │                     │───────────────────►│                      │
 │                     │◄───────────────────│  { authorizationURL }│
 │                     │                    │                      │
 │  ASWebAuthSession   │────────────────────────────────────────────►
 │                     │                    │                      │  User logs in
 │                     │                    │                      │  and consents to
 │                     │                    │                      │  Chat permissions
 │                     │◄────────────────────────────────────────────
 │                     │  Callback with code │                     │
 │                     │                    │                      │
 │                     │  POST /auth/googlechat/exchange { code }   │
 │                     │───────────────────►│                      │
 │                     │                    │  Exchange → tokens   │
 │                     │                    │  Write to container  │
 │                     │◄───────────────────│                      │
 │                     │  ✓ Connected       │                      │
 │◄────────────────────│                    │                      │
```

### iOS implementation

```swift
import AuthenticationServices

class GoogleChatConnectViewController: UIViewController,
    ASWebAuthenticationPresentationContextProviding {

    func startOAuthFlow() async throws {
        var components = URLComponents(string: "https://accounts.google.com/o/oauth2/v2/auth")!
        components.queryItems = [
            URLQueryItem(name: "client_id", value: googleClientID),
            URLQueryItem(name: "redirect_uri", value: "https://openclaw.ai/auth/callback/googlechat"),
            URLQueryItem(name: "response_type", value: "code"),
            URLQueryItem(name: "scope", value: "https://www.googleapis.com/auth/chat.bot"),
            URLQueryItem(name: "access_type", value: "offline"),
            URLQueryItem(name: "state", value: generateCSRFState()),
            URLQueryItem(name: "prompt", value: "consent"),
        ]

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

    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        return view.window!
    }
}
```

### Required scopes

| Scope | Purpose |
| --- | --- |
| `https://www.googleapis.com/auth/chat.bot` | Read and send messages as the bot |
| `https://www.googleapis.com/auth/chat.spaces.readonly` | List spaces the bot is a member of |

---

## Google Chat bot configuration modes

Google Chat supports two delivery modes. Choose based on your infrastructure:

| Mode | How it works | Best for |
| --- | --- | --- |
| **HTTP webhook (App URL)** | Google sends HTTP POST to your endpoint | Managed cloud deployment |
| **Cloud Pub/Sub** | Google publishes to a GCP Pub/Sub topic | High-scale deployments |
| **Direct HTTP** | Bot polls the API | Not recommended; no real-time delivery |

For the OpenClaw managed app, **HTTP webhook** is the recommended mode. Each user's container
registers a unique webhook URL with Google Chat so messages route to the correct container.

---

## Gateway config written to container

```json5
// /data/credentials/googlechat.json  (service account path)
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountKeyFile: "/data/credentials/google-service-account.json",
      projectId: "your-gcp-project",
      dmPolicy: "pairing",
    },
  },
}
```

```json5
// /data/credentials/googlechat.json  (OAuth path)
{
  channels: {
    googlechat: {
      enabled: true,
      accessToken: "ya29.a0...",
      refreshToken: "1//0g...",
      clientId: "…",
      dmPolicy: "pairing",
    },
  },
}
```

## Limitations

- **Workspace only**: Google Chat bots cannot send DMs to `@gmail.com` accounts.
- **Admin approval required**: Depending on Workspace settings, a domain admin may need to
  approve third-party Chat apps before users can add them.
- **Space invitation**: The user must manually add the bot to a Google Chat Space or start a
  DM with it. There is no programmatic way for the bot to initiate contact.

## Related docs

- [Google Chat channel](/channels/googlechat) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/ios-quickconnect)
