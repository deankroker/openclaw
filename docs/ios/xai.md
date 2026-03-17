---
summary: "Implement xAI (Grok) quick connect in the OpenClaw iOS app (API key paste)"
read_when:
  - Adding xAI Grok authentication to the iOS app
  - Implementing xAI API key entry on iOS
title: "xAI (Grok) — iOS Quick Connect"
---

# xAI (Grok) — iOS Quick Connect

xAI provides access to the **Grok** model family via a plain **API key**. The API is
OpenAI-compatible, so it integrates with OpenClaw's existing routing layer without any
additional adapter.

## Option A: API key paste

The user creates a key at [console.x.ai/](https://console.x.ai/) and pastes it into the app.

### iOS implementation

```swift
struct XAIKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your xAI API key")
                .font(.headline)
            SecureField("xai-...", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect Grok") {
                Task { await connect() }
            }
            .disabled(apiKey.trimmingCharacters(in: .whitespaces).isEmpty)
            Link("Get an API key →",
                 destination: URL(string: "https://console.x.ai/")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(
            provider: "xai",
            key: apiKey.trimmingCharacters(in: .whitespaces)
        )
    }
}
```

### Config written to the container

```json5
{
  env: { XAI_API_KEY: "xai-..." },
  agents: {
    defaults: {
      model: { primary: "xai/grok-3" },
    },
  },
}
```

---

## Option C: X (Twitter) iOS app deep link

The **X app** (`twitter://`) is installed on many iOS devices. While X does not expose a
documented Grok API token delegation deep link, you can open the X app to direct users to
their xAI console:

```swift
let xURL = URL(string: "twitter://")!
if UIApplication.shared.canOpenURL(xURL) {
    UIApplication.shared.open(xURL)
} else {
    UIApplication.shared.open(URL(string: "https://console.x.ai/")!)
}
```

Add `twitter` to `LSApplicationQueriesSchemes` in `Info.plist`.

---

## Options B and D

xAI does not currently provide a public OAuth 2.0 authorization endpoint (Option B).

**Option D** (OpenClaw URL scheme) is available for users who have a key on another device:

```
https://app.openclaw.ai/connect?provider=xai&token=<base64-encoded-key>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "xai",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let key = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "xai", key: key) }
}
```

---

## Credential flow to the container

```
User pastes API key
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "xai", "apiKey": "xai-..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  XAI_API_KEY available
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong or expired API key | Regenerate at console.x.ai |
| `model not found` | Model ID incorrect | Use `xai/grok-3` or `xai/grok-3-mini` |
| Rate limit errors | Free-tier quota exceeded | Check usage at console.x.ai and upgrade if needed |

## Related docs

- [iOS Quick Connect overview](/ios)
