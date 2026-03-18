---
summary: "Implement Hugging Face quick connect in the OpenClaw iOS app (access token paste)"
read_when:
  - Adding Hugging Face Inference authentication to the iOS app
  - Implementing Hugging Face token entry on iOS
title: "Hugging Face — iOS Quick Connect"
---

# Hugging Face (Inference) — iOS Quick Connect

Hugging Face Inference Providers expose an OpenAI-compatible router giving access to
DeepSeek, Llama, Qwen, and many more models through a single **fine-grained access token**.
The token must have the **Make calls to Inference Providers** permission enabled.

## Option A: Access token paste

The user creates a fine-grained token at
[huggingface.co/settings/tokens/new](https://huggingface.co/settings/tokens/new?tokenType=fineGrained)
with the **Make calls to Inference Providers** permission and pastes it into the app.

### iOS implementation

```swift
struct HuggingFaceKeyView: View {
    @State private var token = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your Hugging Face access token")
                .font(.headline)
            Text("Required permission: Make calls to Inference Providers")
                .font(.caption)
                .foregroundStyle(.secondary)
            SecureField("hf_...", text: $token)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect Hugging Face") {
                Task { await connect() }
            }
            .disabled(token.isEmpty || !token.hasPrefix("hf_"))
            Link("Create a token →",
                 destination: URL(
                    string: "https://huggingface.co/settings/tokens/new?tokenType=fineGrained"
                 )!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(provider: "huggingface", key: token)
    }
}
```

**Key validation**: Hugging Face tokens start with `hf_`.

### Config written to the container

```json5
{
  env: { HUGGINGFACE_HUB_TOKEN: "hf_..." },
  agents: {
    defaults: {
      model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
    },
  },
}
```

The container accepts either `HUGGINGFACE_HUB_TOKEN` or `HF_TOKEN`.

---

## Options B, C, D

Hugging Face does not currently provide:

- A public OAuth 2.0 authorization endpoint suitable for third-party app sign-in (Option B)
- An iOS app with token delegation (Option C)

**Option D** (OpenClaw URL scheme) is available for users who have a token on another device:

```
https://app.openclaw.ai/connect?provider=huggingface&token=<base64-encoded-token>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "huggingface",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let token = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "huggingface", key: token) }
}
```

---

## Credential flow to the container

```
User pastes access token
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "huggingface", "apiKey": "hf_..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  HUGGINGFACE_HUB_TOKEN available
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong token or missing permission | Recreate token with Inference Providers permission |
| `403 Forbidden` | Model requires Pro subscription | Upgrade HF account or switch to a free model |
| Model list empty in onboarding | Token invalid at discovery time | Verify token and retry |

## Related docs

- [Hugging Face provider (OpenClaw)](/providers/huggingface)
- [iOS Quick Connect overview](./index.md)
