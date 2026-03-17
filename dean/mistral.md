---
summary: "Implement Mistral quick connect in the OpenClaw iOS app (API key paste)"
read_when:
  - Adding Mistral authentication to the iOS app
  - Implementing Mistral API key entry on iOS
title: "Mistral — iOS Quick Connect"
---

# Mistral — iOS Quick Connect

Mistral provides access to its model family (Mistral Large, Codestral, Devstral, and
Voxtral transcription) via a single **API key**. There is no OAuth subscription flow —
the API key paste path is the only supported method.

## Option A: API key paste

The user creates a key at [console.mistral.ai/api-keys](https://console.mistral.ai/api-keys)
and pastes it into the app.

### iOS implementation

```swift
struct MistralKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your Mistral API key")
                .font(.headline)
            SecureField("API key", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect Mistral") {
                Task { await connect() }
            }
            .disabled(apiKey.trimmingCharacters(in: .whitespaces).isEmpty)
            Link("Get an API key →",
                 destination: URL(string: "https://console.mistral.ai/api-keys")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(
            provider: "mistral",
            key: apiKey.trimmingCharacters(in: .whitespaces)
        )
    }
}
```

### Config written to the container

```json5
{
  env: { MISTRAL_API_KEY: "<key>" },
  agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
}
```

For audio transcription with Voxtral, the same key also powers the transcription endpoint:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

---

## Options B, C, D

Mistral does not currently provide:

- A public OAuth 2.0 authorization endpoint for third-party apps (Option B)
- An iOS app with token delegation (Option C)

**Option D** (OpenClaw URL scheme) still applies for users who have a key stored on another
device's gateway:

```
https://app.openclaw.ai/connect?provider=mistral&token=<base64-encoded-key>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "mistral",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let key = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "mistral", key: key) }
}
```

---

## Credential flow to the container

```
User pastes API key
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "mistral", "apiKey": "<key>" }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  MISTRAL_API_KEY available
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong or expired API key | Regenerate at console.mistral.ai |
| `402 Payment Required` | Free-tier credits exhausted | Add payment method in Mistral Console |
| `model_not_found` | Model ID typo | Use `mistral/mistral-large-latest` as a safe default |

## Related docs

- [Mistral provider (OpenClaw)](/providers/mistral)
- [iOS Quick Connect overview](./index.md)
