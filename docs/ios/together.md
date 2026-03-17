---
summary: "Implement Together AI quick connect in the OpenClaw iOS app (API key paste)"
read_when:
  - Adding Together AI authentication to the iOS app
  - Implementing Together API key entry on iOS
title: "Together AI — iOS Quick Connect"
---

# Together AI — iOS Quick Connect

Together AI provides access to leading open-source models (Llama, DeepSeek, Kimi, Qwen,
and more) via a single **API key**. The API is OpenAI-compatible, making it straightforward
to route from OpenClaw.

## Option A: API key paste

The user creates a key at [api.together.ai/settings/api-keys](https://api.together.ai/settings/api-keys)
and pastes it into the app.

### iOS implementation

```swift
struct TogetherKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your Together AI API key")
                .font(.headline)
            SecureField("API key", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect Together AI") {
                Task { await connect() }
            }
            .disabled(apiKey.trimmingCharacters(in: .whitespaces).isEmpty)
            Link("Get an API key →",
                 destination: URL(string: "https://api.together.ai/settings/api-keys")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(
            provider: "together",
            key: apiKey.trimmingCharacters(in: .whitespaces)
        )
    }
}
```

### Config written to the container

```json5
{
  env: { TOGETHER_API_KEY: "<key>" },
  agents: {
    defaults: {
      model: { primary: "together/moonshotai/Kimi-K2.5" },
    },
  },
}
```

Popular model refs for Together AI:

| Model | Ref |
|---|---|
| Kimi K2.5 | `together/moonshotai/Kimi-K2.5` |
| Llama 3.3 70B | `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` |
| DeepSeek R1 | `together/deepseek-ai/DeepSeek-R1` |
| Qwen 2.5 72B | `together/Qwen/Qwen2.5-72B-Instruct-Turbo` |

---

## Options B, C, D

Together AI does not currently provide:

- A public OAuth 2.0 authorization endpoint (Option B)
- An iOS app with token delegation (Option C)

**Option D** (OpenClaw URL scheme) is available for users who have a key on another device:

```
https://app.openclaw.ai/connect?provider=together&token=<base64-encoded-key>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "together",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let key = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "together", key: key) }
}
```

---

## Credential flow to the container

```
User pastes API key
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "together", "apiKey": "<key>" }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  TOGETHER_API_KEY available
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong or expired API key | Regenerate at api.together.ai |
| `402` / credits error | Credit balance depleted | Top up at api.together.ai/billing |
| `model_not_found` | Model ID format incorrect | Check Together AI model catalog for exact ID |

## Related docs

- [Together AI provider (OpenClaw)](/providers/together)
- [iOS Quick Connect overview](/ios)
