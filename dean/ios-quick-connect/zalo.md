---
title: "Zalo – iOS Quick Connect"
summary: "How to connect a Zalo Official Account to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Zalo quick-connect screen in the iOS app
---

# Zalo – iOS Quick Connect

**Auth method:** Token paste  
**Complexity:** Low

Zalo is a Vietnamese messaging platform widely used in Vietnam. OpenClaw connects to it via the **Zalo Official Account (OA)** API using a bot token issued by the Zalo developer platform. The user creates an Official Account, generates an API token, and pastes it into the iOS app.

---

## User flow

```
1. User opens Settings → Channels → Zalo
2. iOS app shows:
   - Link to developers.zalo.me (SFSafariViewController)
   - SecureField for "OA Access Token"
3. User creates a Zalo Official Account and generates an access token
4. Pastes the token and taps "Connect"
5. Container config is patched; gateway connects to Zalo OA webhook
6. Success screen shows the OA name
```

---

## iOS implementation

### 1. UI (SwiftUI sketch)

```swift
struct ZaloConnectView: View {
    @State private var oaAccessToken = ""
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Form {
            Section {
                Link("Open Zalo Developer Portal",
                     destination: URL(string: "https://developers.zalo.me")!)
            } header: {
                Text("Step 1 – Create an Official Account")
            } footer: {
                Text("Create a Zalo Official Account → go to Settings → Permissions → generate an Access Token.")
            }

            Section(header: Text("Step 2 – Paste your OA Access Token")) {
                SecureField("OA Access Token", text: $oaAccessToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
            }

            Section {
                Button("Connect") { Task { await connect() } }
                    .disabled(oaAccessToken.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect Zalo")
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "zalo",
                config: [
                    "enabled": true,
                    "accounts": [
                        "default": ["botToken": oaAccessToken]
                    ],
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

## Zalo OA setup guide (in-app)

The iOS app should include a brief guide (shown as a collapsible card):

1. Go to [developers.zalo.me](https://developers.zalo.me) and sign in.
2. Create a new **Official Account** application.
3. Go to **Settings → Access Token** and generate a token.
4. Copy the token and paste it into the iOS app.

---

## Container config applied

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "<OA_ACCESS_TOKEN>",
        },
      },
      dmPolicy: "pairing",
    },
  },
}
```

---

## Plugin requirement

Zalo ships as a plugin (`@openclaw/zalo`) and must be installed in the container. The iOS app should check plugin status and offer installation:

```http
POST /gateway/plugins/install
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "package": "@openclaw/zalo" }
```

---

## Webhook exposure

Zalo Official Account webhooks require a publicly accessible HTTPS URL. The container must expose its webhook endpoint. Options:

- **Cloudflare Tunnel** (recommended) — The container's gateway port is exposed via Cloudflare Tunnel, and the tunnel URL is registered in the Zalo developer portal.
- **AKS Ingress** — Per-user subdomain.

The iOS app setup screen should display the container's public webhook URL so the user can register it in the Zalo developer portal.

---

## Security notes

- Zalo OA access tokens expire periodically. The iOS app settings screen should surface a "Refresh token" action.
- Store the token only on the container side; do not persist it in the iOS Keychain.
- Webhook endpoints should be authenticated to prevent unauthorized message injection.
