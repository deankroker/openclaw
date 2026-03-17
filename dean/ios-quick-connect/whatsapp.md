---
title: "WhatsApp – iOS Quick Connect"
summary: "How to link a WhatsApp account to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the WhatsApp quick-connect screen in the iOS app
---

# WhatsApp – iOS Quick Connect

**Auth method:** QR scan  
**Complexity:** Medium

OpenClaw uses the WhatsApp Web protocol (Baileys) to link to a WhatsApp account. This is the same mechanism as WhatsApp Web/Desktop: the container generates a QR code and the user scans it with their phone's WhatsApp app. No API keys or developer accounts are required.

> **Note:** The container must be running and reachable for the QR code flow to work. If the container is not yet assigned or healthy, show a loading state and retry.

---

## User flow

```
1. User opens Settings → Channels → WhatsApp
2. iOS app calls the container's link-initiation endpoint
   → Container starts a new Baileys session and returns a QR code (base64 PNG or raw QR data)
3. iOS app renders the QR code
4. User opens WhatsApp on their primary phone:
   Settings → Linked Devices → Link a Device → scan QR
5. Container detects successful link and confirms to the iOS app via SSE or polling
6. iOS app shows success with the linked phone number
```

---

## iOS implementation

### 1. Request a QR code from the container

The file-api pod (or a dedicated container gateway endpoint) generates the QR code:

```swift
struct WhatsAppQRView: View {
    @State private var qrImage: UIImage? = nil
    @State private var status: LinkStatus = .loading
    @State private var pollTask: Task<Void, Never>? = nil

    var body: some View {
        VStack(spacing: 24) {
            if let qr = qrImage {
                Image(uiImage: qr)
                    .interpolation(.none)
                    .resizable()
                    .frame(width: 260, height: 260)
                    .padding()
                    .background(Color.white)
                    .cornerRadius(12)

                Text("Open WhatsApp → Settings → Linked Devices → Link a Device")
                    .font(.footnote)
                    .multilineTextAlignment(.center)
                    .foregroundStyle(.secondary)
            } else if status == .loading {
                ProgressView("Generating QR code…")
            } else if status == .linked {
                Label("WhatsApp linked!", systemImage: "checkmark.circle.fill")
                    .foregroundStyle(.green)
            } else if status == .expired {
                Button("Refresh QR Code") {
                    Task { await loadQR() }
                }
            }
        }
        .navigationTitle("Link WhatsApp")
        .task { await loadQR() }
        .onDisappear { pollTask?.cancel() }
    }

    private func loadQR() async {
        status = .loading
        do {
            let response = try await ContainerService.shared.initiateWhatsAppLink()
            guard let qrData = Data(base64Encoded: response.qrBase64),
                  let image = UIImage(data: qrData) else {
                status = .error
                return
            }
            qrImage = image
            status = .scanning
            pollTask = Task { await pollLinkStatus() }
        } catch {
            status = .error
        }
    }

    private func pollLinkStatus() async {
        // Poll every 3 seconds; QR expires after ~60 seconds
        for _ in 0..<20 {
            try? await Task.sleep(nanoseconds: 3_000_000_000)
            let linked = try? await ContainerService.shared.checkWhatsAppLinkStatus()
            if linked == true {
                status = .linked
                return
            }
        }
        // QR expired
        qrImage = nil
        status = .expired
    }
}
```

### 2. Container endpoints required

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/gateway/whatsapp/link/start` | POST | Starts a new Baileys session and returns a base64 QR code image |
| `/gateway/whatsapp/link/status` | GET | Returns `{ "status": "pending" \| "linked" \| "expired" }` |
| `/gateway/whatsapp/link/cancel` | POST | Cancels the pending link session |

These endpoints are authenticated with the user's Supabase JWT and proxied through the file-api pod.

### 3. QR refresh / expiry

WhatsApp QR codes expire after approximately 60 seconds. The iOS app should:
- Show a countdown timer.
- Automatically request a fresh QR after expiry.
- Limit refresh attempts to avoid rate-limiting by the WhatsApp backend.

---

## Container config applied

WhatsApp does not require a static token config. Once the QR is scanned the Baileys session credentials are stored in the container's `~/.openclaw/sessions/whatsapp/` directory. However, the access policy config should be written:

```json5
{
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: [],        // populated after first approved pairing
    },
  },
}
```

---

## Approving the first message (pairing)

After linking, the first WhatsApp DM to the bot number will generate a pairing request. The iOS app should surface pending pairing requests through a notification or an in-app badge, identical to the Telegram/Discord flows.

---

## Alternative: WhatsApp Business API

For production deployments at scale, consider using the **WhatsApp Cloud API** (Meta) instead of Baileys. This requires:

- A Meta Business account and a verified WhatsApp Business number.
- A permanent API token (System User token).
- A publicly reachable webhook URL for inbound messages.

The iOS quick-connect flow for Cloud API would use **token paste** (permanent system user token + phone number ID), which is lower complexity than the QR scan. See Meta's documentation for [WhatsApp Cloud API setup](https://developers.facebook.com/docs/whatsapp/cloud-api/get-started).

---

## Security notes

- The linked WhatsApp session is equivalent to a logged-in WhatsApp Web session. It grants access to the account's full message history.
- Encourage users to use a dedicated WhatsApp number (not their personal number) to avoid accidental message interception.
- The container should enforce `allowFrom` to whitelist only authorized phone numbers.
- QR code endpoint responses must be scoped to the authenticated user's container only; never expose another user's QR data.
