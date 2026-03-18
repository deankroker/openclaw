---
title: "iMessage (BlueBubbles) – iOS Quick Connect"
summary: "How to connect iMessage via a BlueBubbles server to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the iMessage / BlueBubbles quick-connect screen in the iOS app
---

# iMessage (BlueBubbles) – iOS Quick Connect

**Auth method:** Guided form (server URL + password)  
**Complexity:** Medium

iMessage is supported through the [BlueBubbles](https://bluebubbles.app) macOS helper app. The user must have a Mac running the BlueBubbles server accessible from the internet (or via a tunnel). The iOS app connects the user's OpenClaw container to that server using the server's URL and password.

> **Prerequisite:** The user must already have a Mac running macOS 13+ with the BlueBubbles server installed and its web API enabled. This is an inherently macOS-dependent integration.

---

## User flow

```
1. User opens Settings → Channels → iMessage
2. iOS app shows a "Connect BlueBubbles" screen with:
   - Text field for "Server URL" (e.g., https://my-mac.example.com:1234)
   - SecureField for "Password"
   - (Optional) text field for "Webhook path" (default: /bluebubbles-webhook)
3. User taps "Test Connection" → app verifies /health endpoint
4. User taps "Connect" → config is written to the container
5. Container registers its webhook with the BlueBubbles server
6. iOS app shows success with server version info
```

---

## iOS implementation

### 1. UI (SwiftUI sketch)

```swift
struct BlueBubblesConnectView: View {
    @State private var serverURL = ""
    @State private var password = ""
    @State private var webhookPath = "/bluebubbles-webhook"
    @State private var healthResult: String? = nil
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Form {
            Section(header: Text("BlueBubbles Server")) {
                TextField("https://my-mac.example.com:1234", text: $serverURL)
                    .textContentType(.URL)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
                SecureField("Password", text: $password)
                    .textContentType(.password)
            }
            Section(header: Text("Webhook Path")) {
                TextField("/bluebubbles-webhook", text: $webhookPath)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }
            Section {
                Button("Test Connection") { Task { await testConnection() } }
                if let health = healthResult {
                    Text(health).foregroundStyle(.green).font(.footnote)
                }
            }
            Section {
                Button("Connect") { Task { await connect() } }
                    .disabled(serverURL.isEmpty || password.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect iMessage")
    }

    private func testConnection() async {
        guard let url = URL(string: serverURL.appending("/api/v1/ping")) else { return }
        var request = URLRequest(url: url)
        request.setValue(password, forHTTPHeaderField: "x-password")
        guard let (data, _) = try? await URLSession.shared.data(for: request) else {
            healthResult = "✗ Could not reach server"
            return
        }
        let ping = try? JSONDecoder().decode(BlueBubblesPing.self, from: data)
        healthResult = ping != nil ? "✓ Connected to BlueBubbles \(ping?.data.name ?? "")" : "✗ Could not reach server"
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "bluebubbles",
                config: [
                    "enabled": true,
                    "serverUrl": serverURL,
                    "password": password,
                    "webhookPath": webhookPath,
                ]
            )
            status = .connected
        } catch {
            status = .error(error.localizedDescription)
        }
    }
}
```

### 2. Connection test (client-side)

The iOS app can directly call the BlueBubbles `/api/v1/ping` endpoint before submitting to the container:

```swift
func testBlueBubblesServer(url: String, password: String) async throws -> BlueBubblesPing {
    var request = URLRequest(url: URL(string: "\(url)/api/v1/ping")!)
    request.setValue(password, forHTTPHeaderField: "x-password")
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw ChannelError.serverUnreachable
    }
    return try JSONDecoder().decode(BlueBubblesPing.self, from: data)
}
```

---

## Container config applied

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      serverUrl: "https://my-mac.example.com:1234",
      password: "<BLUEBUBBLES_PASSWORD>",
      webhookPath: "/bluebubbles-webhook",
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
}
```

After the config is applied the container will automatically register its webhook URL with the BlueBubbles server. The user should not need to do anything in the BlueBubbles Mac app.

---

## Webhook registration (automatic)

When the OpenClaw gateway starts with a valid BlueBubbles config it registers a webhook with the server at:

```
https://<gateway-public-url>/bluebubbles-webhook?password=<password>
```

Since OpenClaw containers run behind a ClusterIP-only service, the gateway must be accessible to the BlueBubbles server. Options:

- **Cloudflare Tunnel** — Recommended. The container's gateway port is exposed via a dedicated Cloudflare Tunnel, and the tunnel URL is registered as the webhook.
- **ngrok / other tunnels** — Acceptable for development.
- **Direct AKS ingress** — Requires additional AKS ingress configuration.

The iOS app setup screen can display the tunnel URL for the user's container so they can verify webhook registration in the BlueBubbles app.

---

## BlueBubbles server setup guide (in-app)

The iOS app should include a brief setup guide for users who have not yet configured BlueBubbles on their Mac:

1. Download and install [BlueBubbles for macOS](https://bluebubbles.app/install).
2. Open BlueBubbles → Settings → Server → enable **Enable LAN URL**.
3. Set a strong **Server Password**.
4. Expose the server to the internet (port forwarding, Cloudflare Tunnel, or ngrok).
5. Copy the public URL and paste it into the iOS app.

---

## Security notes

- The BlueBubbles password grants full iMessage read/write access. Treat it like a root credential.
- Only accept HTTPS server URLs; reject plain HTTP to prevent token interception.
- Webhook authentication is always verified by OpenClaw using the `x-password` / `?password=` parameter before processing any webhook payload.
- Recommend users run BlueBubbles on a dedicated Mac account with iMessage signed in only for the bot number.
