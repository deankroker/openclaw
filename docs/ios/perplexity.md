---
summary: "Implement Perplexity quick connect in the OpenClaw iOS app (API key paste)"
read_when:
  - Adding Perplexity authentication to the iOS app
  - Implementing Perplexity API key entry on iOS
title: "Perplexity — iOS Quick Connect"
---

# Perplexity — iOS Quick Connect

Perplexity provides real-time web search augmented AI responses through the
`llama-3.1-sonar-*` model family. Access requires a plain **API key** from the Perplexity
platform.

## Option A: API key paste

The user creates a key at [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)
and pastes it into the app.

### iOS implementation

```swift
struct PerplexityKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your Perplexity API key")
                .font(.headline)
            SecureField("pplx-...", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect Perplexity") {
                Task { await connect() }
            }
            .disabled(apiKey.trimmingCharacters(in: .whitespaces).isEmpty)
            Link("Get an API key →",
                 destination: URL(string: "https://www.perplexity.ai/settings/api")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(
            provider: "perplexity",
            key: apiKey.trimmingCharacters(in: .whitespaces)
        )
    }
}
```

### Config written to the container

OpenClaw routes Perplexity through the Brave Search / Perplexity integration. The container
needs the key exposed as `PERPLEXITY_API_KEY`:

```json5
{
  env: { PERPLEXITY_API_KEY: "pplx-..." },
}
```

When Perplexity is configured as the web-search provider, the agent automatically enriches
responses with real-time search results.

---

## Option C: Perplexity iOS app deep link

The **Perplexity iOS app** is available on the App Store (`ai.perplexity.app`). It does not
expose a documented OAuth delegation deep link for third-party apps, but you can open it:

```swift
let perplexityURL = URL(string: "perplexity://")!
if UIApplication.shared.canOpenURL(perplexityURL) {
    UIApplication.shared.open(perplexityURL)
} else {
    UIApplication.shared.open(
        URL(string: "https://apps.apple.com/app/perplexity-ask-anything/id1668000334")!
    )
}
```

Add `perplexity` to `LSApplicationQueriesSchemes` in `Info.plist`.

---

## Options B and D

Perplexity does not currently provide a public OAuth 2.0 authorization endpoint (Option B).

**Option D** (OpenClaw URL scheme) is available for users who have a key on another device:

```
https://app.openclaw.ai/connect?provider=perplexity&token=<base64-encoded-key>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "perplexity",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let key = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "perplexity", key: key) }
}
```

---

## Credential flow to the container

```
User pastes API key
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "perplexity", "apiKey": "pplx-..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  PERPLEXITY_API_KEY available
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong or expired API key | Regenerate at perplexity.ai/settings/api |
| Search results not included | Perplexity tool not activated in container config | Ensure Perplexity is configured as the search provider |
| `402` / credits error | Credits depleted | Top up at perplexity.ai/settings/api |

## Related docs

- [Perplexity tool (OpenClaw)](/perplexity)
- [iOS Quick Connect overview](/ios)
