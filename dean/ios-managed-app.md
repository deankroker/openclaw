---
summary: "Per-provider guide to implementing quick connect in a managed OpenClaw iOS app"
read_when:
  - You are building an iOS app that manages OpenClaw containers on behalf of users
  - You need to implement provider authentication from a native iOS interface
  - You want per-provider quick-connect patterns (API key, OAuth, setup-token, AWS credentials, endpoint)
title: "iOS Managed App — Provider Quick Connect"
---

# iOS Managed App — Provider Quick Connect

This guide covers how to implement **quick connect** for every supported model provider in an iOS app that manages dedicated OpenClaw containers (e.g., hosted on Azure Kubernetes Service). Users never run the OpenClaw CLI themselves — the iOS app controls their container configuration through the [file API](/install/ios-managed-app#the-file-api-bridge).

## Architecture recap

```
┌─────────────────┐    ┌──────────────┐    ┌──────────────────────────┐
│                 │    │              │    │  Azure Cloud (AKS)       │
│   iOS App       │◄──►│  Supabase    │    │                          │
│  (SwiftUI)      │    │  (Auth + DB) │    │  ┌────────────────────┐  │
│                 │    │              │    │  │  OpenClaw Pod       │  │
│ • Provider UI   │    │ • Users      │    │  │  (ClusterIP only)   │  │
│ • OAuth sheets  │    │ • Containers │    │  └─────────┬──────────┘  │
│ • Key input     │    │ • RLS        │    │            │ Azure Files │
│                 │    │              │    │  ┌─────────┴──────────┐  │
└────────┬────────┘    └──────────────┘    │  │  file-api pod       │  │
         │                                 │  │  (Cloudflare Tunnel)│  │
         │  HTTPS + Supabase JWT           │  └────────────────────┘  │
         └──▶ files.spark.ooo ─────────────┘                          │
                                           └──────────────────────────┘
```

## The file-API bridge

The file-api pod (`https://files.spark.ooo`) is the control plane between the iOS app and the user's OpenClaw container. It:

- Authenticates every request using the Supabase JWT from the iOS app.
- Reads and writes files in the user's mounted Azure Files share (the same share OpenClaw mounts).
- Exposes endpoints for browsing, reading, writing, and deleting files — including `openclaw.json` (the OpenClaw gateway config).

### Writing provider credentials from iOS

All provider credentials follow the same pattern:

1. **Collect** the credential in the iOS app UI (text field, OAuth callback, etc.).
2. **Read** the current `openclaw.json` from the file API.
3. **Merge** the new provider config into the JSON (preserve all other config keys).
4. **Write** the updated `openclaw.json` back to the file API.
5. **Signal** the container to reload its config (e.g., `POST /restart` or `PUT /config/reload`, depending on your file-api implementation).

```swift
// Conceptual Swift helper — adapt to your actual file-api client
func saveProviderKey(envKey: String, value: String) async throws {
    var config = try await fileAPI.readOpenClawConfig()
    config.env[envKey] = value
    try await fileAPI.writeOpenClawConfig(config)
    try await fileAPI.reloadGateway()
}
```

Config is written to the user's `openclaw.json` as environment variables under the `env` key:

```json5
{
  env: {
    OPENAI_API_KEY: "sk-...",
    ANTHROPIC_API_KEY: "sk-ant-...",
    // ...
  },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Connection types

| Type | Providers | iOS pattern |
|------|-----------|-------------|
| [API key paste](#api-key-providers) | OpenAI, Anthropic, Mistral, Together AI, OpenRouter, Venice, Hugging Face, MiniMax, Moonshot, NVIDIA, GLM, Qianfan, Kilocode, Xiaomi, Z.AI, Deepgram, Vercel AI Gateway | `SecureField` + file-api write |
| [OAuth via browser sheet](#oauth-providers) | GitHub Copilot, OpenAI Codex, Qwen | `ASWebAuthenticationSession` |
| [Device flow](#device-flow-providers) | GitHub Copilot (fallback) | Show code + URL in app |
| [Setup-token paste](#anthropic-claude-subscription--setup-token) | Anthropic (Claude subscription) | `SecureField` + file-api write |
| [AWS credentials](#amazon-bedrock) | Amazon Bedrock | Two `SecureField`s + file-api write |
| [Endpoint URL](#endpoint-based-providers) | Ollama, vLLM, sglang, LiteLLM, Cloudflare AI Gateway | `TextField` + file-api write |

---

## API key providers

For providers that authenticate with a single secret token, the flow is:

1. Show a `SecureField` (masked input) for the key.
2. Provide a **"Get your key"** button that opens the provider's dashboard in `SFSafariViewController`.
3. On submit, write `env.PROVIDER_API_KEY` via the file API.
4. Confirm with a brief success banner.

```swift
struct APIKeyConnectView: View {
    let envKey: String        // e.g. "OPENAI_API_KEY"
    let dashboardURL: URL
    @State private var apiKey = ""

    var body: some View {
        Form {
            Section("API key") {
                SecureField("Paste your key", text: $apiKey)
            }
            Section {
                Button("Get your key") {
                    UIApplication.shared.open(dashboardURL)
                }
                Button("Connect") {
                    Task { try await saveProviderKey(envKey: envKey, value: apiKey) }
                }
                .disabled(apiKey.isEmpty)
            }
        }
    }
}
```

### OpenAI

- **Dashboard:** `https://platform.openai.com/api-keys`
- **Env key:** `OPENAI_API_KEY`
- **Key format:** `sk-...`
- **Recommended model:** `openai/gpt-5.4`

```swift
APIKeyConnectView(
    envKey: "OPENAI_API_KEY",
    dashboardURL: URL(string: "https://platform.openai.com/api-keys")!
)
```

Config written to `openclaw.json`:

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic (API key)

- **Dashboard:** `https://console.anthropic.com/settings/keys`
- **Env key:** `ANTHROPIC_API_KEY`
- **Key format:** `sk-ant-...`
- **Recommended model:** `anthropic/claude-opus-4-6`

Config written to `openclaw.json`:

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

See also [Anthropic Claude subscription (setup-token)](#anthropic-claude-subscription--setup-token) if the user has a Claude subscription instead of an API key.

### Mistral

- **Dashboard:** `https://console.mistral.ai/api-keys/`
- **Env key:** `MISTRAL_API_KEY`
- **Key format:** `sk-...`
- **Recommended model:** `mistral/mistral-large-latest`

### Together AI

- **Dashboard:** `https://api.together.xyz/settings/api-keys`
- **Env key:** `TOGETHER_API_KEY`
- **Recommended model:** `together/moonshotai/Kimi-K2.5`

### OpenRouter

- **Dashboard:** `https://openrouter.ai/keys`
- **Env key:** `OPENROUTER_API_KEY`
- **Key format:** `sk-or-...`
- **Recommended model:** `openrouter/anthropic/claude-sonnet-4-5`

### Venice AI

- **Dashboard:** `https://venice.ai/settings/api`
- **Env key:** `VENICE_API_KEY`
- **Recommended model:** `venice/llama-3.3-70b`

### Hugging Face

- **Dashboard:** `https://huggingface.co/settings/tokens`
- **Token type:** Fine-grained, with **Make calls to Inference Providers** permission
- **Env key:** `HUGGINGFACE_HUB_TOKEN`
- **Recommended model:** `huggingface/meta-llama/Llama-3.3-70B-Instruct`

### MiniMax

- **Dashboard:** `https://platform.minimax.chat/user-center/basic-information/interface-key`
- **Env key:** `MINIMAX_API_KEY`

### Moonshot AI (Kimi)

- **Dashboard:** `https://platform.moonshot.cn/console/api-keys`
- **Env key:** `MOONSHOT_API_KEY`
- **Recommended model:** `moonshot/moonshot-v1-8k`

### NVIDIA

- **Dashboard:** `https://build.nvidia.com/`
- **Env key:** `NVIDIA_API_KEY`
- **Key format:** `nvapi-...`

### GLM (Zhipu AI)

- **Dashboard:** `https://open.bigmodel.cn/usercenter/apikeys`
- **Env key:** `GLM_API_KEY`

### Qianfan (Baidu)

Qianfan requires both a key and a secret:

- **Dashboard:** `https://console.bce.baidu.com/qianfan/ais/console/applicationConsole/application`
- **Env keys:** `QIANFAN_API_KEY` and `QIANFAN_SECRET_KEY`

For Qianfan, show two `SecureField`s — one for the Access Key and one for the Secret Key — and write both to `env` in `openclaw.json`.

### Kilocode

- **Dashboard:** `https://app.kilo.ai`
- **Env key:** `KILOCODE_API_KEY`
- **Recommended model:** `kilocode/kilo/auto`

### Xiaomi AI

- **Dashboard:** `https://ai.xiaomi.com/`
- **Env key:** `XIAOMI_API_KEY`

### Z.AI

- **Dashboard:** `https://z.ai/`
- **Env key:** `ZAI_API_KEY`

### Deepgram (transcription)

- **Dashboard:** `https://console.deepgram.com/`
- **Env key:** `DEEPGRAM_API_KEY`
- **Key format:** `Token ...`

### Vercel AI Gateway

- **Dashboard:** `https://vercel.com/`
- **Env key:** `VERCEL_API_KEY`

---

## OAuth providers

For OAuth providers, the iOS app opens an in-app browser sheet via `ASWebAuthenticationSession`. After the user authorizes in the browser, the provider redirects to a deep-link URL registered by the iOS app, which carries the token or authorization code. The app then exchanges the code (if needed) and writes the resulting token to the file API.

### Deep-link registration

Register a custom URL scheme (e.g., `openclaw://auth/callback`) in your iOS app's `Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>openclaw</string>
    </array>
  </dict>
</array>
```

Use this callback URL when constructing OAuth authorize URLs.

---

### OpenAI Codex (ChatGPT subscription)

OpenAI supports OAuth for ChatGPT/Codex subscribers. The iOS app initiates the standard Authorization Code + PKCE flow.

**Endpoints:**
- Authorize: `https://auth.openai.com/oauth/authorize`
- Token: `https://auth.openai.com/oauth/token`

**Scopes:** `openid profile email offline_access`

**Flow:**

```swift
import AuthenticationServices

func connectOpenAICodex() async throws -> String {
    let codeVerifier = generatePKCEVerifier()
    let codeChallenge = sha256Base64URL(codeVerifier)

    var components = URLComponents(string: "https://auth.openai.com/oauth/authorize")!
    components.queryItems = [
        .init(name: "response_type", value: "code"),
        .init(name: "client_id",     value: YOUR_OPENAI_CLIENT_ID),
        .init(name: "redirect_uri",  value: "openclaw://auth/callback"),
        .init(name: "scope",         value: "openid profile email offline_access"),
        .init(name: "code_challenge", value: codeChallenge),
        .init(name: "code_challenge_method", value: "S256"),
    ]

    let code = try await ASWebAuthenticationSession.authenticate(
        url: components.url!,
        callbackURLScheme: "openclaw"
    )
    let accessToken = try await exchangeCodeForToken(code, verifier: codeVerifier,
        tokenURL: "https://auth.openai.com/oauth/token")

    // Write the Codex token via file API
    var config = try await fileAPI.readOpenClawConfig()
    config.setCodexToken(accessToken)   // writes to models.json / auth store
    try await fileAPI.writeOpenClawConfig(config)
    return accessToken
}
```

Tokens are stored in the container's auth profile store (not as plain env vars). The container needs a matching `models.providers.openai-codex` entry pointing at the Codex endpoint.

**Recommended model after auth:** `openai-codex/gpt-5.4`

---

### GitHub Copilot (OAuth — recommended)

GitHub uses OAuth 2.0 with the device authorization grant (device flow) or with a standard authorization code flow for apps registered in GitHub OAuth Apps.

#### Authorization code flow (best for iOS)

Register your iOS app as a GitHub OAuth App at `https://github.com/settings/developers`. Set the callback URL to `openclaw://auth/callback`.

```swift
func connectGitHubCopilot() async throws {
    var components = URLComponents(string: "https://github.com/login/oauth/authorize")!
    components.queryItems = [
        .init(name: "client_id", value: YOUR_GITHUB_CLIENT_ID),
        .init(name: "scope",     value: "read:user"),
        .init(name: "redirect_uri", value: "openclaw://auth/callback"),
    ]

    let code = try await ASWebAuthenticationSession.authenticate(
        url: components.url!,
        callbackURLScheme: "openclaw"
    )
    let githubToken = try await exchangeGitHubCode(code)

    // OpenClaw exchanges this GitHub token for a Copilot API token at runtime.
    // Store the GitHub token in the auth profile store via file API.
    var config = try await fileAPI.readOpenClawConfig()
    config.setGitHubToken(githubToken)
    try await fileAPI.writeOpenClawConfig(config)
}
```

**Recommended model after auth:** `github-copilot/gpt-4o` or `github-copilot/gpt-4.1`

---

### GitHub Copilot (device flow — fallback)

If you cannot register a GitHub OAuth App (e.g., enterprise restrictions), use the device authorization grant. The iOS app polls until the user completes authorization in their browser.

```swift
struct DeviceFlowConnectView: View {
    @State private var userCode: String = ""
    @State private var verificationURL: URL?
    @State private var isPolling = false

    var body: some View {
        VStack(spacing: 20) {
            if let url = verificationURL {
                Text("Open this URL in your browser:")
                Link(url.absoluteString, destination: url)
                Text("Then enter this code:")
                    .font(.headline)
                Text(userCode)
                    .font(.largeTitle.monospaced())
                    .padding()
                    .background(Color(.secondarySystemBackground))
                    .cornerRadius(8)
                Button("Open in Safari") {
                    UIApplication.shared.open(url)
                }
            } else {
                ProgressView("Starting device flow...")
            }
        }
        .task { await startDeviceFlow() }
    }

    func startDeviceFlow() async {
        let response = try! await requestDeviceCodes(
            clientID: YOUR_GITHUB_CLIENT_ID,
            scope: "read:user"
        )
        userCode = response.userCode
        verificationURL = URL(string: response.verificationURL)
        isPolling = true
        let token = try! await pollForToken(deviceCode: response.deviceCode,
                                            interval: response.interval)
        // Store token via file API
    }
}
```

---

### Qwen (OAuth)

Qwen uses a device-code OAuth flow through the Qwen Portal.

1. Call the Qwen device-code endpoint to get a user code and verification URL.
2. Display the code to the user (same `DeviceFlowConnectView` pattern above).
3. Poll until the user completes auth on the Qwen Portal website.
4. Store the resulting access token and refresh token via the file API.

```swift
// After obtaining the Qwen token:
var config = try await fileAPI.readOpenClawConfig()
config.setQwenToken(accessToken: token.accessToken, refreshToken: token.refreshToken)
try await fileAPI.writeOpenClawConfig(config)
```

The Qwen plugin (`qwen-portal-auth`) must be enabled on the OpenClaw container before the token is useful. Enable it via the file API by writing the plugin config:

```json5
{
  plugins: { enabled: ["qwen-portal-auth"] },
}
```

**Recommended models after auth:**
- `qwen-portal/coder-model`
- `qwen-portal/vision-model`

---

## Anthropic Claude subscription — setup-token

Users with a Claude subscription (Pro, Team, or Enterprise) can authenticate using a **setup-token** generated by the Claude Code CLI. This is a paste-based flow — the user generates the token on a desktop, copies it, then pastes it in the iOS app.

**UI pattern:**

1. Show a `SecureField` for the setup-token.
2. Show instructions: _"On your Mac or PC, install the Claude CLI and run `claude setup-token`. Copy the token and paste it here."_
3. On submit, store the token via the file API in the container's auth profile store (not as a plain env var — setup-tokens have a dedicated storage path).

```swift
struct ClaudeSetupTokenView: View {
    @State private var token = ""

    var body: some View {
        Form {
            Section("Setup-token") {
                SecureField("Paste setup-token", text: $token)
            }
            Section {
                VStack(alignment: .leading, spacing: 8) {
                    Text("How to get a setup-token:")
                        .font(.headline)
                    Text("1. Install the Claude CLI on your Mac or PC.")
                    Text("2. Run `claude setup-token` in a terminal.")
                    Text("3. Copy the token that appears.")
                    Text("4. Paste it above.")
                }
                .font(.callout)
                .foregroundStyle(.secondary)
            }
            Section {
                Button("Connect") {
                    Task { try await saveSetupToken(token) }
                }
                .disabled(token.isEmpty)
            }
        }
    }
}
```

The setup-token is written to the container's `models.auth.profiles` store (via file API) rather than to `env`. The container's OpenClaw process reads it at startup and uses it to refresh OAuth tokens automatically.

**Recommended model after auth:** `anthropic/claude-opus-4-6`

---

## Amazon Bedrock

Amazon Bedrock uses AWS credentials rather than a simple API key. For a managed iOS app, the safest approach is to collect the user's **AWS Access Key ID** and **AWS Secret Access Key** (optionally with a **Session Token** for temporary credentials) and store them in the container's environment.

<Warning>
Do not use long-lived IAM user credentials for production deployments. Prefer IAM roles with restricted Bedrock permissions attached to the AKS pod service account via IRSA or Azure Workload Identity Federation. If you must collect credentials from the iOS app, use short-lived credentials generated by AWS STS or your own credential broker.
</Warning>

**UI pattern:**

```swift
struct BedrockConnectView: View {
    @State private var accessKeyID = ""
    @State private var secretAccessKey = ""
    @State private var sessionToken = ""
    @State private var region = "us-east-1"

    var body: some View {
        Form {
            Section("AWS Credentials") {
                TextField("Access Key ID", text: $accessKeyID)
                    .autocorrectionDisabled()
                SecureField("Secret Access Key", text: $secretAccessKey)
                SecureField("Session Token (optional)", text: $sessionToken)
            }
            Section("Region") {
                TextField("AWS Region", text: $region)
            }
            Section {
                Button("Connect") {
                    Task { try await saveBedrockCredentials() }
                }
                .disabled(accessKeyID.isEmpty || secretAccessKey.isEmpty)
            }
        }
    }

    func saveBedrockCredentials() async throws {
        var config = try await fileAPI.readOpenClawConfig()
        config.env["AWS_ACCESS_KEY_ID"] = accessKeyID
        config.env["AWS_SECRET_ACCESS_KEY"] = secretAccessKey
        if !sessionToken.isEmpty {
            config.env["AWS_SESSION_TOKEN"] = sessionToken
        }
        config.env["AWS_REGION"] = region
        try await fileAPI.writeOpenClawConfig(config)
        try await fileAPI.reloadGateway()
    }
}
```

Config written to `openclaw.json`:

```json5
{
  env: {
    AWS_ACCESS_KEY_ID: "AKIA...",
    AWS_SECRET_ACCESS_KEY: "...",
    AWS_REGION: "us-east-1",
  },
  agents: {
    defaults: {
      model: { primary: "amazon-bedrock/anthropic.claude-opus-4-6-20250514-v1:0" },
    },
  },
}
```

---

## Endpoint-based providers

For providers that run inside (or alongside) the user's container, the iOS app only needs to configure an endpoint URL — no secret is required.

### Ollama (in-container)

Ollama can run as a sidecar container alongside the OpenClaw pod. From the container's perspective, it is reachable at `http://127.0.0.1:11434`.

**UI pattern:** A read-only status screen showing the Ollama endpoint and available models. The user does not configure credentials — models are managed server-side.

Config written to `openclaw.json`:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://127.0.0.1:11434",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "ollama/llama3.2" },
    },
  },
}
```

If users can choose which Ollama model to use, present the model list (fetched from the file API or a container status endpoint) as a picker in the iOS settings screen.

### vLLM

vLLM exposes an OpenAI-compatible API, typically on port 8000.

```json5
{
  env: { VLLM_API_KEY: "vllm-local" },
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "vllm/meta-llama/Llama-3.1-8B-Instruct" },
    },
  },
}
```

### sglang

```json5
{
  env: { SGLANG_API_KEY: "sglang-local" },
  models: {
    providers: {
      sglang: {
        baseUrl: "http://127.0.0.1:30000/v1",
      },
    },
  },
}
```

### LiteLLM proxy

If the managed service runs a shared LiteLLM proxy that routes to multiple backends:

- **Env key:** `LITELLM_API_KEY` (or the proxy's configured master key)
- **Env key:** `LITELLM_BASE_URL` (the proxy URL)

```json5
{
  env: {
    LITELLM_API_KEY: "sk-...",
    LITELLM_BASE_URL: "https://litellm.your-infra.example.com",
  },
}
```

### Cloudflare AI Gateway

Cloudflare AI Gateway is a managed proxy/cache that sits in front of other providers. In the iOS app, configure the gateway URL and the underlying provider key together.

- **Dashboard:** `https://dash.cloudflare.com` → AI → AI Gateway
- **Env key:** `CLOUDFLARE_AI_GATEWAY_URL` (the gateway's base URL)
- Plus the underlying provider key (e.g., `OPENAI_API_KEY`)

```json5
{
  env: {
    CLOUDFLARE_AI_GATEWAY_URL: "https://gateway.ai.cloudflare.com/v1/ACCOUNT/GATEWAY_NAME",
    OPENAI_API_KEY: "sk-...",
  },
}
```

---

## Provider picker UX

Combine all of the above into a single **"Add provider"** flow in the iOS app:

```swift
struct AddProviderView: View {
    enum Provider: String, CaseIterable, Identifiable {
        case anthropicKey    = "Anthropic (API key)"
        case anthropicToken  = "Anthropic (Claude subscription)"
        case openaiKey       = "OpenAI (API key)"
        case openaiCodex     = "OpenAI (Codex / ChatGPT)"
        case githubCopilot   = "GitHub Copilot"
        case qwen            = "Qwen"
        case bedrock         = "Amazon Bedrock"
        case mistral         = "Mistral"
        case togetherAI      = "Together AI"
        case openrouter      = "OpenRouter"
        case venice          = "Venice AI"
        case huggingface     = "Hugging Face"
        case ollama          = "Ollama (in-container)"
        case vllm            = "vLLM (in-container)"
        var id: String { rawValue }
    }

    @State private var selectedProvider: Provider?

    var body: some View {
        List(Provider.allCases) { provider in
            Button(provider.rawValue) {
                selectedProvider = provider
            }
        }
        .navigationTitle("Add provider")
        .navigationDestination(item: $selectedProvider) { provider in
            destinationView(for: provider)
        }
    }

    @ViewBuilder
    func destinationView(for provider: Provider) -> some View {
        switch provider {
        case .anthropicKey:
            APIKeyConnectView(
                envKey: "ANTHROPIC_API_KEY",
                dashboardURL: URL(string: "https://console.anthropic.com/settings/keys")!
            )
        case .anthropicToken:
            ClaudeSetupTokenView()
        case .openaiKey:
            APIKeyConnectView(
                envKey: "OPENAI_API_KEY",
                dashboardURL: URL(string: "https://platform.openai.com/api-keys")!
            )
        case .openaiCodex:
            OAuthConnectView(provider: .openaiCodex)
        case .githubCopilot:
            OAuthConnectView(provider: .githubCopilot)
        case .qwen:
            DeviceFlowConnectView(provider: .qwen)
        case .bedrock:
            BedrockConnectView()
        case .mistral:
            APIKeyConnectView(
                envKey: "MISTRAL_API_KEY",
                dashboardURL: URL(string: "https://console.mistral.ai/api-keys/")!
            )
        case .togetherAI:
            APIKeyConnectView(
                envKey: "TOGETHER_API_KEY",
                dashboardURL: URL(string: "https://api.together.xyz/settings/api-keys")!
            )
        case .openrouter:
            APIKeyConnectView(
                envKey: "OPENROUTER_API_KEY",
                dashboardURL: URL(string: "https://openrouter.ai/keys")!
            )
        case .venice:
            APIKeyConnectView(
                envKey: "VENICE_API_KEY",
                dashboardURL: URL(string: "https://venice.ai/settings/api")!
            )
        case .huggingface:
            APIKeyConnectView(
                envKey: "HUGGINGFACE_HUB_TOKEN",
                dashboardURL: URL(string: "https://huggingface.co/settings/tokens")!
            )
        case .ollama:
            EndpointStatusView(providerName: "Ollama", endpoint: "http://127.0.0.1:11434")
        case .vllm:
            EndpointStatusView(providerName: "vLLM", endpoint: "http://127.0.0.1:8000/v1")
        }
    }
}
```

---

## Security considerations

- **Never log or transmit API keys in plain text.** Use HTTPS everywhere — the file API already requires Supabase JWT over TLS.
- **Mask keys in the UI** after they are saved. Show only the last 4 characters (e.g., `sk-ant-...abcd`).
- **Validate key format client-side** before submission (e.g., `sk-` prefix for OpenAI) to catch paste errors early.
- **Scope OAuth tokens** to the minimum required permissions. For GitHub Copilot, `read:user` is enough.
- **Rotate secrets** via the same Settings → Provider screen. Overwriting a key via the file API replaces the old value atomically.
- **AWS credentials**: prefer short-lived STS credentials and a dedicated IAM role with only `bedrock:InvokeModel*` permissions. Do not store root account credentials.

## See also

- [OpenAI](/providers/openai)
- [Anthropic](/providers/anthropic)
- [GitHub Copilot](/providers/github-copilot)
- [Qwen](/providers/qwen)
- [Amazon Bedrock](/providers/bedrock)
- [Ollama](/providers/ollama)
- [vLLM](/providers/vllm)
- [Model providers overview](/providers/index)
