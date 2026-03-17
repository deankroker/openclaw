---
summary: "Implement OpenAI GPT / Codex quick connect in the OpenClaw iOS app (API key, ChatGPT OAuth, ChatGPT app deep link)"
read_when:
  - Adding OpenAI authentication to the iOS app
  - Implementing ChatGPT OAuth (PKCE) or API-key flows for GPT models
title: "OpenAI — iOS Quick Connect"
---

# OpenAI (GPT / Codex) — iOS Quick Connect

OpenAI supports two credential types: a plain **API key** (`sk-proj-...` or `sk-...`) and
**ChatGPT / Codex OAuth** (subscription sign-in). OpenAI explicitly supports OAuth subscription
usage in external tools. API key access uses the `/v1/chat/completions` endpoint;
Codex subscription uses a separate Codex OAuth flow.

## Option A: API key paste

The user creates a key at [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
and pastes it into the app.

### iOS implementation

```swift
struct OpenAIKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your OpenAI API key")
                .font(.headline)
            SecureField("sk-...", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect OpenAI") {
                Task { await connect() }
            }
            .disabled(apiKey.isEmpty || !apiKey.hasPrefix("sk-"))
            Link("Get an API key →",
                 destination: URL(string: "https://platform.openai.com/api-keys")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(provider: "openai", key: apiKey)
    }
}
```

**Key validation**: OpenAI API keys start with `sk-`. Project keys use `sk-proj-`.

### Config written to the container

```json5
{
  env: { OPENAI_API_KEY: "sk-proj-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

---

## Option B: ChatGPT / Codex OAuth PKCE browser flow

OpenAI provides an OAuth 2.0 flow for ChatGPT and Codex subscription access. Use
`ASWebAuthenticationSession` with PKCE to run the browser handshake inside the app.

### PKCE helper

```swift
import CryptoKit

struct PKCEChallenge {
    let verifier: String
    let challenge: String

    init() {
        let verifierData = (0..<32).map { _ in UInt8.random(in: 0...255) }
        verifier = Data(verifierData).base64URLEncoded()
        let hash = SHA256.hash(data: Data(verifier.utf8))
        challenge = Data(hash).base64URLEncoded()
    }
}

extension Data {
    func base64URLEncoded() -> String {
        base64EncodedString()
            .replacingOccurrences(of: "+", with: "-")
            .replacingOccurrences(of: "/", with: "_")
            .trimmingCharacters(in: CharacterSet(charactersIn: "="))
    }
}
```

### OAuth flow

```swift
import AuthenticationServices

class OpenAIAuthCoordinator: NSObject, ASWebAuthenticationPresentationContextProviding {
    private let clientID = "<your-openai-oauth-client-id>"
    private let redirectURI = "https://app.openclaw.ai/oauth/openai/callback"

    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        UIApplication.shared.connectedScenes
            .compactMap { $0 as? UIWindowScene }
            .flatMap { $0.windows }
            .first { $0.isKeyWindow } ?? ASPresentationAnchor()
    }

    func startOAuth() async throws -> String {
        let pkce = PKCEChallenge()
        var components = URLComponents(string: "https://auth.openai.com/authorize")!
        components.queryItems = [
            .init(name: "response_type",    value: "code"),
            .init(name: "client_id",        value: clientID),
            .init(name: "redirect_uri",     value: redirectURI),
            .init(name: "scope",            value: "openid email"),
            .init(name: "code_challenge",   value: pkce.challenge),
            .init(name: "code_challenge_method", value: "S256"),
        ]

        let callbackURL = try await withCheckedThrowingContinuation { continuation in
            let auth = ASWebAuthenticationSession(
                url: components.url!,
                callbackURLScheme: "https"
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

        // Exchange code for token via your backend / file-api relay
        return try await exchangeCode(from: callbackURL, verifier: pkce.verifier)
    }
}
```

**Important**: Exchange the authorization code for tokens **server-side** (through the
file-api or a dedicated backend endpoint) to keep `client_secret` out of the iOS binary.

---

## Option C: ChatGPT iOS app deep link

The **ChatGPT iOS app** (`chatgpt://`) does not currently expose a documented OAuth delegation
deep link for third-party apps. You can still open it for reference:

```swift
let chatGPTURL = URL(string: "chatgpt://")!
if UIApplication.shared.canOpenURL(chatGPTURL) {
    UIApplication.shared.open(chatGPTURL)
} else {
    UIApplication.shared.open(
        URL(string: "https://apps.apple.com/app/chatgpt/id6448311069")!
    )
}
```

Add `chatgpt` to `LSApplicationQueriesSchemes` in `Info.plist`.

Monitor the [OpenAI platform changelog](https://platform.openai.com/docs/changelog) for
any first-party mobile OAuth delegation.

---

## Option D: OpenClaw URL scheme quick auth

For users who ran `openclaw models auth login --provider openai-codex` on their gateway,
the token can be transferred to the iOS app via a universal link:

```
https://app.openclaw.ai/connect?provider=openai&token=<base64-encoded-token>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "openai",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let token = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "openai", key: token) }
}
```

---

## Credential flow to the container

```
User pastes API key / OAuth token arrives
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "openai", "apiKey": "sk-proj-..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  OPENAI_API_KEY available
```

For Codex subscription, write `OPENAI_CODEX_TOKEN` (or the appropriate env var) instead.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401` from OpenAI | Wrong or expired key | Regenerate at platform.openai.com |
| `insufficient_quota` | Free-tier exhausted or billing not set up | Add payment method in OpenAI dashboard |
| OAuth code exchange fails | `redirect_uri` mismatch | Verify URI matches OpenAI app registration |
| Codex model returns `model_not_found` | Model requires Codex subscription | Ensure Codex plan is active |

## Related docs

- [OpenAI provider (OpenClaw)](/providers/openai)
- [iOS Quick Connect overview](/ios)
