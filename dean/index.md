---
summary: "How to implement quick connect for AI providers in the OpenClaw managed iOS app"
read_when:
  - Building the OpenClaw iOS app provider connection UI
  - Choosing an auth strategy for a specific AI provider
  - Implementing OAuth, deep links, or API-key flows on iOS
title: "iOS Quick Connect"
---

# iOS Quick Connect — Provider Integration Guide

This section describes how to implement **quick connect** flows for each AI provider in the
OpenClaw managed iOS app. The app uses Supabase for user auth, runs OpenClaw in a dedicated
AKS container per user, and exposes file/config operations through the file-api
(`https://files.spark.ooo`).

## What is quick connect?

Quick connect is any flow that lets a user link their AI provider account to their OpenClaw
container without leaving the app for longer than necessary. Four levels of friction exist,
from highest control to lowest:

| Option | Friction | Provider support |
|---|---|---|
| [API key / token paste](#option-a-api-key--token-paste) | Low dev effort, highest user friction | All providers |
| [OAuth PKCE browser flow](#option-b-oauth-pkce-browser-flow) | Medium dev effort, smooth UX | Anthropic, OpenAI, Google, GitHub |
| [Provider native app deep link](#option-c-provider-native-app-deep-link) | Low user friction if app is installed | Anthropic (Claude app), OpenAI (ChatGPT app), Google (Gemini app) |
| [OpenClaw URL scheme](#option-d-openclaw-url-scheme-quick-auth) | Lowest friction for returning users | First-party only |

## How credentials reach the container

Regardless of connection method, the end goal is the same: store the provider credential
so the user's OpenClaw container can use it.

```
iOS App  →  Supabase JWT  →  file-api (files.spark.ooo)
                              ↕ Azure Files mount
                         OpenClaw Pod (AKS, ClusterIP)
```

The file-api authenticates every request with the Supabase JWT (`Authorization: Bearer <jwt>`).
After the iOS app obtains a provider credential, write it to the container config:

```
PUT https://files.spark.ooo/config
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "provider": "anthropic", "apiKey": "sk-ant-..." }
```

The container reloads its config and picks up the new credential. See the file-api docs for
the exact schema and reload semantics.

## Option A: API key / token paste

The simplest path. The user copies their API key from the provider's web console and pastes
it into the app.

### SwiftUI implementation

```swift
struct APIKeyEntryView: View {
    @State private var apiKey = ""

    var body: some View {
        SecureField("Paste API key", text: $apiKey)
            .textContentType(.password)
            .autocorrectionDisabled()
            .textInputAutocapitalization(.never)
        Button("Connect") {
            Task { await saveKey(apiKey) }
        }
    }

    func saveKey(_ key: String) async {
        // 1. Validate key client-side (non-empty, prefix check)
        // 2. POST to file-api with Supabase JWT
        // 3. Show success / error
    }
}
```

**Security note**: Never log or store the raw key in `UserDefaults`. Use the Keychain for any
local caching, and prefer sending the key directly to the file-api over HTTPS so it does not
persist on-device longer than necessary.

## Option B: OAuth PKCE browser flow

For providers that support OAuth 2.0 with PKCE, use `ASWebAuthenticationSession` to run the
browser handshake without leaving the app. The system presents a Safari View Controller in-app;
no external browser switch occurs.

### Generic Swift scaffolding

```swift
import AuthenticationServices

func startOAuth(
    authURL: URL,
    callbackScheme: String
) async throws -> URL {
    try await withCheckedThrowingContinuation { continuation in
        let session = ASWebAuthenticationSession(
            url: authURL,
            callbackURLScheme: callbackScheme
        ) { callbackURL, error in
            if let error { continuation.resume(throwing: error); return }
            guard let url = callbackURL else {
                continuation.resume(throwing: OAuthError.missingCallback)
                return
            }
            continuation.resume(returning: url)
        }
        session.presentationContextProvider = self
        session.prefersEphemeralWebBrowserSession = false // keep cookies for SSO
        session.start()
    }
}
```

Exchange the authorization code for a token server-side (or via the file-api relay) to keep
the client secret out of the iOS binary.

## Option C: Provider native app deep link

If the provider has an iOS app that exposes an OAuth deep link, you can delegate auth to that
app and receive the result back via a universal link or custom URL scheme. This eliminates the
browser sheet entirely for users who already have the provider app installed.

Use `UIApplication.shared.canOpenURL(_:)` to check availability before surfacing this option.

```swift
func connectViaProviderApp(url: URL, fallback: () -> Void) {
    if UIApplication.shared.canOpenURL(url) {
        UIApplication.shared.open(url)
    } else {
        fallback() // fall back to browser OAuth or key paste
    }
}
```

Register the corresponding universal link domain in your entitlements and handle the incoming
URL in `onOpenURL`:

```swift
.onOpenURL { url in
    handleProviderCallback(url)
}
```

## Option D: OpenClaw URL scheme quick auth

For users who have previously authenticated a provider on another device or through the
gateway CLI, OpenClaw can expose its own URL scheme so the token is transferred in-app:

```
openclaw://connect?provider=anthropic&token=<base64-encoded-token>
```

The receiving app validates the token, writes it to the file-api, and shows a confirmation.
Deep links should use Universal Links (`https://`) in production to prevent scheme hijacking.

---

## Per-provider docs

- [Anthropic (Claude)](./anthropic.md)
- [OpenAI (GPT / Codex)](./openai.md)
- [Google (Gemini)](./google.md)
- [GitHub Copilot](./github-copilot.md)
- [Mistral](./mistral.md)
- [OpenRouter](./openrouter.md)
- [Together AI](./together.md)
- [Hugging Face](./huggingface.md)
- [xAI (Grok)](./xai.md)
- [Perplexity](./perplexity.md)
