---
title: "Feishu / Lark – iOS Quick Connect"
summary: "How to connect a Feishu or Lark bot to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Feishu / Lark quick-connect screen in the iOS app
---

# Feishu / Lark – iOS Quick Connect

**Auth method:** Token paste (App ID + App Secret)  
**Complexity:** Low

Feishu (known as Lark outside China) is a team communication platform by ByteDance. OpenClaw connects to it through a custom Feishu app. The user creates an app on the Feishu developer platform and provides its **App ID** and **App Secret**. No OAuth round-trip is required.

---

## User flow

```
1. User opens Settings → Channels → Feishu
2. iOS app shows:
   - Link: "Open Feishu Developer Console" → open.feishu.cn (or open.larksuite.com for Lark)
   - Text field: App ID
   - SecureField: App Secret
3. User taps "Connect"
4. Container config is patched; gateway subscribes to events via WebSocket
5. Success screen shows app name and verification status
```

---

## iOS implementation

### 1. UI (SwiftUI sketch)

```swift
struct FeishuConnectView: View {
    @State private var appId = ""
    @State private var appSecret = ""
    @State private var isLark = false       // Lark (international) vs Feishu (China)
    @State private var status: ConnectionStatus = .idle

    var devConsoleURL: URL {
        isLark
            ? URL(string: "https://open.larksuite.com/app")!
            : URL(string: "https://open.feishu.cn/app")!
    }

    var body: some View {
        Form {
            Section {
                Toggle("Use Lark (international)", isOn: $isLark)
                Link("Open Developer Console", destination: devConsoleURL)
            } header: {
                Text("Step 1 – Create a Feishu app")
            } footer: {
                Text("Go to the developer console → Create App → Custom App. Copy the App ID and App Secret from the Credentials section.")
            }

            Section(header: Text("Step 2 – Enter credentials")) {
                TextField("cli_xxxxxxxxxxxxxxxx", text: $appId)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
                SecureField("App Secret", text: $appSecret)
                    .textContentType(.password)
            }

            Section {
                Button("Connect") { Task { await connect() } }
                    .disabled(appId.isEmpty || appSecret.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect Feishu")
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "feishu",
                config: [
                    "enabled": true,
                    "appId": appId,
                    "appSecret": appSecret,
                    "platform": isLark ? "lark" : "feishu",
                ]
            )
            status = .connected
        } catch {
            status = .error(error.localizedDescription)
        }
    }
}
```

### 2. Token validation (optional)

The Feishu API has a tenant access token endpoint that can be used to validate credentials before writing them to the container:

```swift
func validateFeishuCredentials(appId: String, appSecret: String, isLark: Bool) async throws {
    let baseURL = isLark ? "https://open.larksuite.com" : "https://open.feishu.cn"
    let url = URL(string: "\(baseURL)/open-apis/auth/v3/app_access_token/internal")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json; charset=utf-8", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(["app_id": appId, "app_secret": appSecret])

    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw ChannelError.invalidToken
    }
    struct TokenResponse: Decodable { let code: Int; let msg: String }
    let result = try JSONDecoder().decode(TokenResponse.self, from: data)
    guard result.code == 0 else {
        throw ChannelError.custom(result.msg)
    }
}
```

---

## Feishu app setup guide (in-app)

The iOS app should include a brief guide (shown as a collapsible card or sheet):

1. Open [Feishu Developer Console](https://open.feishu.cn/app) (or [Lark](https://open.larksuite.com/app)).
2. Click **Create App → Custom App**.
3. Choose **Self-Built App**, give it a name (e.g., "OpenClaw").
4. Go to **Credentials & Basic Info** → copy **App ID** and **App Secret**.
5. Go to **Event Subscriptions** → choose **Long Connection** (WebSocket). No public URL needed.
6. Add required permissions: `im:message`, `im:message.receive_v1`.
7. Publish the app to your workspace.

---

## Container config applied

```json5
{
  channels: {
    feishu: {
      enabled: true,
      appId: "cli_xxxxxxxxxxxxxxxx",
      appSecret: "<APP_SECRET>",
      dmPolicy: "pairing",
    },
  },
}
```

---

## Plugin requirement

Feishu ships bundled with current OpenClaw releases. No separate plugin installation is required for containers running a recent image.

If the container image is older, the iOS app can trigger a plugin installation:

```http
POST /gateway/plugins/install
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "package": "@openclaw/feishu" }
```

---

## Security notes

- App Secrets grant full access to the Feishu bot. Treat them like passwords and never log them.
- Use event subscription mode (Long Connection / WebSocket) rather than webhook mode so no public URL is required, reducing the attack surface.
- Scope the Feishu app's permissions to the minimum required (`im:message` and `im:message.receive_v1`).
