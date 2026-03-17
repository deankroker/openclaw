---
title: "LINE – iOS Quick Connect"
summary: "How to connect a LINE Messaging API channel to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the LINE quick-connect screen in the iOS app
---

# LINE – iOS Quick Connect

**Auth method:** Token paste (channel access token + channel secret)  
**Complexity:** Low

LINE uses the Messaging API with a **channel access token** and **channel secret**. The user creates a Messaging API channel on the LINE Developers Console and pastes the two credentials into the iOS app. A publicly accessible webhook URL is also required.

---

## User flow

```
1. User opens Settings → Channels → LINE
2. iOS app shows:
   - Link: "Open LINE Developers Console" (SFSafariViewController)
   - SecureField: "Channel Access Token"
   - SecureField: "Channel Secret"
   - Display-only: Container's public webhook URL
3. User creates a Messaging API channel, copies credentials, enables webhook
4. Pastes credentials into the iOS app and taps "Connect"
5. Container config is patched; gateway starts listening for LINE events
6. App shows success with bot name
```

---

## iOS implementation

### 1. UI (SwiftUI sketch)

```swift
struct LineConnectView: View {
    @State private var accessToken = ""
    @State private var channelSecret = ""
    @State private var status: ConnectionStatus = .idle

    var webhookURL: String {
        (ContainerService.shared.containerInfo?.webhookBaseURL ?? "https://…") + "/line/webhook"
    }

    var body: some View {
        Form {
            Section {
                Link("Open LINE Developers Console",
                     destination: URL(string: "https://developers.line.biz/console/")!)
            } header: {
                Text("Step 1 – Create a Messaging API channel")
            } footer: {
                Text("Create a Provider → New Channel → Messaging API. Copy the Channel Access Token and Channel Secret.")
            }

            Section(header: Text("Channel Access Token")) {
                SecureField("Long-lived access token", text: $accessToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
            }

            Section(header: Text("Channel Secret")) {
                SecureField("32-char hex secret", text: $channelSecret)
                    .textContentType(.password)
                    .autocorrectionDisabled()
            }

            Section {
                Text("Webhook URL (paste into LINE Console):")
                    .font(.footnote)
                    .foregroundStyle(.secondary)
                Text(webhookURL)
                    .font(.footnote.monospaced())
                Button("Copy Webhook URL") { UIPasteboard.general.string = webhookURL }
            } header: {
                Text("Step 2 – Configure webhook in LINE Console")
            } footer: {
                Text("In the LINE Console, go to Messaging API → Webhook settings, paste the URL above, and enable 'Use webhook'.")
            }

            Section {
                Button("Connect") { Task { await connect() } }
                    .disabled(accessToken.isEmpty || channelSecret.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect LINE")
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "line",
                config: [
                    "enabled": true,
                    "channelAccessToken": accessToken,
                    "channelSecret": channelSecret,
                ]
            )
            status = .connected
        } catch {
            status = .error(error.localizedDescription)
        }
    }
}
```

---

## LINE Developers Console setup guide (in-app)

1. Go to [developers.line.biz/console](https://developers.line.biz/console).
2. Create or select a **Provider**.
3. Click **Create a new channel** → choose **Messaging API**.
4. Fill in the required fields and create the channel.
5. Go to **Messaging API** tab:
   - Under **Channel access token**, click **Issue** to generate a long-lived token.
   - Copy the **Channel secret** from the **Basic settings** tab.
6. Under **Webhook settings**:
   - Set Webhook URL to the container's webhook URL shown in the iOS app.
   - Enable **Use webhook**.
   - Click **Verify** to confirm the endpoint is reachable.
7. Disable the **Auto-reply messages** and **Greeting messages** in the LINE Official Account Manager to prevent double responses.

---

## Webhook exposure

LINE requires a publicly accessible HTTPS endpoint. Each user's container must expose its `/line/webhook` endpoint via:

- **Cloudflare Tunnel** (recommended) — The container's gateway port is exposed via Cloudflare Tunnel.
- **AKS Ingress** — Per-user subdomain.

---

## Container config applied

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "<ACCESS_TOKEN>",
      channelSecret: "<CHANNEL_SECRET>",
      dmPolicy: "pairing",
      webhookPath: "/line/webhook",
    },
  },
}
```

---

## Plugin requirement

LINE ships as a plugin (`@openclaw/line`) and must be installed in the container. The iOS app should check plugin status and offer installation:

```http
POST /gateway/plugins/install
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "package": "@openclaw/line" }
```

---

## Security notes

- LINE uses HMAC-SHA256 signature verification on webhook payloads. OpenClaw verifies every incoming webhook using the channel secret.
- Store the channel access token and channel secret only on the container side.
- LINE access tokens can be revoked from the LINE Developers Console if the integration needs to be disabled.
