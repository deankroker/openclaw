---
summary: "Implement Google Gemini quick connect in the OpenClaw iOS app (API key, Google OAuth, Gemini app deep link)"
read_when:
  - Adding Google Gemini authentication to the iOS app
  - Implementing Google Sign-In or AI Studio API key flows
title: "Google (Gemini) — iOS Quick Connect"
---

# Google (Gemini) — iOS Quick Connect

Google provides access to Gemini models via **AI Studio API keys** (simple bearer token)
and via **Google OAuth 2.0** (for workspace or advanced quota). For most consumer iOS users,
the AI Studio API key path is the fastest to ship and the most reliable.

## Option A: AI Studio API key paste

The user creates a key at [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey)
and pastes it into the app.

### iOS implementation

```swift
struct GoogleKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your Google AI Studio API key")
                .font(.headline)
            SecureField("AIza...", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect Gemini") {
                Task { await connect() }
            }
            .disabled(apiKey.isEmpty || !apiKey.hasPrefix("AIza"))
            Link("Get an API key →",
                 destination: URL(string: "https://aistudio.google.com/app/apikey")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(provider: "google", key: apiKey)
    }
}
```

**Key validation**: Google AI Studio keys start with `AIza`.

### Config written to the container

```json5
{
  env: { GEMINI_API_KEY: "AIza..." },
  agents: { defaults: { model: { primary: "google/gemini-2.5-pro" } } },
}
```

---

## Option B: Google OAuth 2.0 PKCE browser flow

For workspace accounts or users who want OAuth-based access (e.g., Vertex AI via ADC),
use `ASWebAuthenticationSession` with Google's authorization endpoint.

### PKCE flow with Google

```swift
import AuthenticationServices
import CryptoKit

class GoogleAuthCoordinator: NSObject, ASWebAuthenticationPresentationContextProviding {
    private let clientID = "<your-google-oauth-client-id>"
    // Use the reverse-DNS form of your client ID as the callback scheme for installed apps:
    // e.g. "com.googleusercontent.apps.<clientid>"
    private let redirectScheme = "com.googleusercontent.apps.<clientid>"
    private let redirectURI: String

    override init() {
        redirectURI = "\(redirectScheme):/oauth2callback"
        super.init()
    }

    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        UIApplication.shared.connectedScenes
            .compactMap { $0 as? UIWindowScene }
            .flatMap { $0.windows }
            .first { $0.isKeyWindow } ?? ASPresentationAnchor()
    }

    func startOAuth() async throws -> String {
        let pkce = PKCEChallenge()
        var components = URLComponents(string: "https://accounts.google.com/o/oauth2/v2/auth")!
        components.queryItems = [
            .init(name: "response_type",         value: "code"),
            .init(name: "client_id",             value: clientID),
            .init(name: "redirect_uri",          value: redirectURI),
            .init(name: "scope",                 value: "https://www.googleapis.com/auth/generative-language.retriever openid email"),
            .init(name: "code_challenge",        value: pkce.challenge),
            .init(name: "code_challenge_method", value: "S256"),
            .init(name: "access_type",           value: "offline"),
            .init(name: "prompt",                value: "consent"),
        ]

        let callbackURL = try await withCheckedThrowingContinuation { continuation in
            let auth = ASWebAuthenticationSession(
                url: components.url!,
                callbackURLScheme: redirectScheme
            ) { url, error in
                if let error { continuation.resume(throwing: error); return }
                guard let url else {
                    continuation.resume(throwing: OAuthError.missingCallback)
                    return
                }
                continuation.resume(returning: url)
            }
            auth.presentationContextProvider = self
            auth.prefersEphemeralWebBrowserSession = false
            auth.start()
        }

        // Exchange code → access + refresh token via file-api relay
        return try await exchangeCode(from: callbackURL, verifier: pkce.verifier)
    }
}
```

**Scopes**:

| Use case | Scope |
|---|---|
| Gemini API (AI Studio) | `https://www.googleapis.com/auth/generative-language.retriever` |
| Vertex AI | `https://www.googleapis.com/auth/cloud-platform` |
| User identity | `openid email` |

**Client type**: Create an **iOS app** OAuth 2.0 client ID in Google Cloud Console (not a web
client). iOS clients use the reverse-DNS scheme (`com.googleusercontent.apps.<id>`:/oauth2callback)
as the redirect URI and do not require a client secret.

Exchange the authorization code for tokens **server-side** (via the file-api relay) to avoid
storing secrets in the app. The container can refresh the access token using the stored
refresh token.

---

## Option C: Google / Gemini iOS app deep link

The **Google app** and **Gemini app** (`com.google.GoogleMobile`, `com.google.Gemini`) do not
expose documented OAuth delegation deep links for third-party token transfer.

You can still direct users to the Gemini app for reference:

```swift
let geminiURL = URL(string: "gemini://")!
if UIApplication.shared.canOpenURL(geminiURL) {
    UIApplication.shared.open(geminiURL)
} else {
    UIApplication.shared.open(
        URL(string: "https://apps.apple.com/app/google-gemini/id6477489729")!
    )
}
```

Add `gemini` to `LSApplicationQueriesSchemes` in `Info.plist`.

---

## Option D: OpenClaw URL scheme quick auth

For users who authenticated Google/Gemini on a desktop gateway, transfer the credential:

```
https://app.openclaw.ai/connect?provider=google&token=<base64-encoded-key>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "google",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let key = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "google", key: key) }
}
```

---

## Credential flow to the container

```
User pastes AI Studio key / OAuth access token arrives
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "google", "apiKey": "AIza..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  GEMINI_API_KEY available
```

For OAuth-based access, store the refresh token in the container and configure the container
to refresh the access token on startup.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `API key not valid` | Wrong key | Regenerate at aistudio.google.com |
| `PERMISSION_DENIED` | Key restricted to wrong project / API | Check API restrictions in Google Cloud Console |
| OAuth consent screen blocks | OAuth app not published | Submit app for verification or use test users |
| `invalid_client` during token exchange | Wrong client ID or redirect URI | Verify OAuth client type is "iOS" in Cloud Console |

## Related docs

- [iOS Quick Connect overview](/ios)
