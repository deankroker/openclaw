---
title: "Tlon / Urbit – iOS Quick Connect"
summary: "How to connect an Urbit ship (Tlon) to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Tlon / Urbit quick-connect screen in the iOS app
---

# Tlon / Urbit – iOS Quick Connect

**Auth method:** Guided form (ship URL + login code)  
**Complexity:** Medium

Tlon is a decentralized messenger built on Urbit. OpenClaw connects to an Urbit ship using the ship's URL and login code (`+code` from the Dojo). The user must already have a running Urbit ship accessible from the internet (or via a hosted service like Red Horizon).

---

## User flow

```
1. User opens Settings → Channels → Tlon
2. iOS app shows a form:
   - Text field: "Ship Name" (e.g., ~sampel-palnet)
   - Text field: "Ship URL" (e.g., https://sampel-palnet.arvo.network)
   - SecureField: "Login Code" (from Dojo: +code)
   - Text field: "Owner Ship" (your personal ship, always allowed DMs, optional)
3. User taps "Connect"
4. Container config is patched; gateway authenticates with the ship
5. Success screen shows ship name and connection status
```

---

## iOS implementation

### 1. Getting the login code

The iOS app should explain how to obtain the login code:

```
In your Urbit Dojo (terminal), run:
  +code
This prints a 4-word code like: lidlut-tabwed-pillex-ridrup
```

Alternatively, if the user has the Tlon iOS app installed, they can copy the code from their ship's profile settings.

### 2. UI (SwiftUI sketch)

```swift
struct TlonConnectView: View {
    @State private var shipName = ""        // e.g. ~sampel-palnet
    @State private var shipURL = ""         // e.g. https://sampel-palnet.arvo.network
    @State private var loginCode = ""       // lidlut-tabwed-pillex-ridrup
    @State private var ownerShip = ""       // optional: your personal ~ship
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Form {
            Section(header: Text("Ship Name")) {
                TextField("~sampel-palnet", text: $shipName)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }

            Section(header: Text("Ship URL")) {
                TextField("https://sampel-palnet.arvo.network", text: $shipURL)
                    .textContentType(.URL)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }

            Section(header: Text("Login Code")) {
                SecureField("lidlut-tabwed-pillex-ridrup", text: $loginCode)
                    .textContentType(.password)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            } footer: {
                Text("Run +code in your Urbit Dojo or copy from your ship's settings.")
            }

            Section(header: Text("Owner Ship (optional)")) {
                TextField("~your-main-ship", text: $ownerShip)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            } footer: {
                Text("Your personal ship will always be allowed to DM the bot.")
            }

            Section {
                Button("Connect") { Task { await connect() } }
                    .disabled(shipName.isEmpty || shipURL.isEmpty || loginCode.isEmpty)
            }
        }
        .navigationTitle("Connect Tlon")
    }

    private func connect() async {
        status = .connecting
        var config: [String: Any] = [
            "enabled": true,
            "ship": shipName,
            "url": shipURL,
            "code": loginCode,
            "dmPolicy": "pairing",
        ]
        if !ownerShip.isEmpty { config["ownerShip"] = ownerShip }
        do {
            try await ChannelConfigService.shared.patch(channel: "tlon", config: config)
            status = .connected
        } catch {
            status = .error(error.localizedDescription)
        }
    }
}
```

---

## Hosting options (in-app guide)

For users who do not yet have a running Urbit ship, the iOS app can link to hosted options:

| Provider | URL |
|----------|-----|
| Red Horizon (Tlon) | [tlon.io](https://tlon.io) |
| Tirrel | [tirrel.io](https://tirrel.io) |
| Self-hosted | [urbit.org/getting-started](https://urbit.org/getting-started) |

---

## Container config applied

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://sampel-palnet.arvo.network",
      code: "<LOGIN_CODE>",
      ownerShip: "~your-main-ship",     // optional
      dmPolicy: "pairing",
    },
  },
}
```

---

## Private / LAN ships

If the ship is running on a private network (localhost, LAN, or internal hostname), the container config must explicitly opt in to allow private network access:

```json5
{
  channels: {
    tlon: {
      url: "http://192.168.1.100:8080",
      allowPrivateNetwork: true,
    },
  },
}
```

The iOS app should warn the user that this configuration may not be reachable from the cloud container if their ship is on a local network.

---

## Plugin requirement

Tlon ships as a plugin (`@openclaw/tlon`) and must be installed in the container. The iOS app should check plugin status and offer installation:

```http
POST /gateway/plugins/install
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "package": "@openclaw/tlon" }
```

---

## Security notes

- The Urbit login code (`+code`) is equivalent to a root password for the ship. Treat it as a secret.
- For the OpenClaw container to reach the ship, the ship must be publicly accessible (or reachable from the container's network). HTTPS is strongly recommended.
- The login code can be rotated by running `|code %reset` in the Urbit Dojo. Update the container config after rotating.
- Do not store the login code in the iOS Keychain after the initial setup POST; it lives only on the container.
