---
title: "Nostr – iOS Quick Connect"
summary: "How to connect a Nostr identity to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Nostr quick-connect screen in the iOS app
---

# Nostr – iOS Quick Connect

**Auth method:** Key paste or in-app key generation  
**Complexity:** Low

Nostr is a decentralized, open protocol for censorship-resistant messaging. OpenClaw connects to Nostr via NIP-04 encrypted DMs using a secp256k1 keypair. The iOS app can either:

1. **Generate a new keypair** in-app (no setup required by the user).
2. **Import an existing private key** (hex or `nsec` bech32 format).

---

## User flow

### New key (zero-config path)

```
1. User opens Settings → Channels → Nostr
2. iOS app shows two options: "Generate new key" and "Import existing key"
3. User taps "Generate new key"
4. iOS app generates a random 32-byte key and displays:
   - The npub (public key, bech32) — user can share this as their bot's address
   - The nsec (private key) — masked, with a "Copy" and "Save to Notes" button
5. App sends the private key to the container
6. Container config is patched with optional relay URLs
7. Success screen shows the bot's npub
```

### Existing key

```
1. User selects "Import existing key"
2. Paste field for nsec (bech32) or hex private key
3. App derives and displays the matching npub for confirmation
4. Taps "Connect"
```

---

## iOS implementation

### 1. Key generation

```swift
import CryptoKit

struct NostrKeyPair {
    let privateKeyHex: String
    let publicKeyHex: String

    static func generate() -> NostrKeyPair {
        // Generate a random 32-byte secret
        var bytes = [UInt8](repeating: 0, count: 32)
        _ = SecRandomCopyBytes(kSecRandomDefault, 32, &bytes)
        let privateKeyHex = bytes.map { String(format: "%02x", $0) }.joined()
        // Derive the secp256k1 public key.
        // Use a library such as https://github.com/GigaBitcoin/secp256k1.swift
        // or https://github.com/nicktindall/swift-nostr for this step.
        let publicKeyHex = Secp256k1.derivePublicKey(from: privateKeyHex)
        return NostrKeyPair(privateKeyHex: privateKeyHex, publicKeyHex: publicKeyHex)
    }
}
```

### 2. UI (SwiftUI sketch)

```swift
struct NostrConnectView: View {
    @State private var mode: Mode = .choose
    @State private var privateKey = ""      // hex or nsec
    @State private var derivedNpub = ""
    @State private var relayURLs = "wss://relay.damus.io,wss://relay.primal.net"
    @State private var status: ConnectionStatus = .idle

    enum Mode { case choose, generate, importKey }

    var body: some View {
        Form {
            if mode == .choose {
                Section {
                    Button("Generate a new key") {
                        let kp = NostrKeyPair.generate()
                        privateKey = kp.privateKeyHex
                        // bech32Encode: use a Swift bech32 library (e.g. https://github.com/0xfair/bech32-swift)
                        // to encode the hex pubkey with hrp "npub".
                        derivedNpub = bech32Encode(hrp: "npub", data: kp.publicKeyHex)
                        mode = .generate
                    }
                    Button("Import existing key") { mode = .importKey }
                }
            } else {
                if mode == .generate {
                    Section(header: Text("Your bot's public key (npub) — share this")) {
                        Text(derivedNpub)
                            .font(.footnote.monospaced())
                        Button("Copy npub") { UIPasteboard.general.string = derivedNpub }
                    }
                    Section(header: Text("Private key — keep secret")) {
                        Text(String(repeating: "•", count: 32))
                        Button("Copy nsec (private key)") {
                            UIPasteboard.general.string = privateKey
                        }
                        .foregroundStyle(.orange)
                    }
                } else {
                    Section(header: Text("Private Key (nsec or hex)")) {
                        SecureField("nsec1... or 64-char hex", text: $privateKey)
                            .textContentType(.password)
                            .autocorrectionDisabled()
                            .onChange(of: privateKey) { _ in
                                // deriveNpubFromKey: decode the nsec bech32 to hex, then derive
                                // the secp256k1 public key and bech32-encode it as npub.
                                derivedNpub = deriveNpubFromKey(privateKey)
                            }
                        if !derivedNpub.isEmpty {
                            Text("npub: \(derivedNpub)").font(.footnote).foregroundStyle(.secondary)
                        }
                    }
                }

                Section(header: Text("Relay URLs (comma-separated)")) {
                    TextField("wss://relay.damus.io", text: $relayURLs)
                        .autocorrectionDisabled()
                        .autocapitalization(.none)
                }

                Section {
                    Button("Connect") { Task { await connect() } }
                        .disabled(privateKey.isEmpty || status == .connecting)
                }
            }
        }
        .navigationTitle("Connect Nostr")
    }

    private func connect() async {
        status = .connecting
        let relays = relayURLs.split(separator: ",").map(String.init)
        do {
            try await ChannelConfigService.shared.patch(
                channel: "nostr",
                config: [
                    "enabled": true,
                    "privateKey": privateKey,
                    "relayUrls": relays,
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

## Container config applied

```json5
{
  channels: {
    nostr: {
      enabled: true,
      privateKey: "<32-byte-hex-or-nsec>",
      relayUrls: [
        "wss://relay.damus.io",
        "wss://relay.primal.net",
      ],
      dmPolicy: "pairing",
    },
  },
}
```

---

## Plugin requirement

Nostr ships as a plugin (`@openclaw/nostr`) and must be installed in the container before enabling the channel. The iOS app should check plugin status and offer installation:

```http
POST /gateway/plugins/install
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "package": "@openclaw/nostr" }
```

---

## Key management in the iOS app

- After the initial setup, the iOS app should **not** store the private key locally. The key lives only on the container.
- The settings screen can display the npub (public key) fetched from the container for reference.
- Provide a "Rotate key" action that generates a new keypair and updates the container config. Warn the user that rotating the key changes their bot's Nostr identity.

---

## Security notes

- A Nostr private key is the user's entire identity on the protocol; it cannot be revoked or changed without creating a new identity.
- The key should be transmitted to the container only over HTTPS and stored only in the container's isolated Azure Files volume.
- For the generated-key flow, display a clear warning that losing the key means losing the identity permanently.
