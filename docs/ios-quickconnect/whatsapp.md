---
summary: "Quick connect WhatsApp from the managed iOS app using QR scan or phone-link deep link"
read_when:
  - Implementing WhatsApp channel setup in the iOS app
  - Understanding QR link and phone-number link flows for WhatsApp Web
title: "WhatsApp — iOS Quick Connect"
sidebarTitle: "WhatsApp"
---

# WhatsApp — iOS Quick Connect

**Pattern: QR code scan via embedded WebView or phone-link deep link**

WhatsApp uses Baileys (WhatsApp Web protocol), which requires the user to link a device via
QR code — the same flow they use to link WhatsApp Web in a browser. In a managed iOS app,
this QR code is rendered inside a `WKWebView` pointed at a lightweight endpoint served by the
user's OpenClaw container.

<Note>
WhatsApp does not have an official bot or business API for individual users. This flow uses
the WhatsApp Web (Baileys) protocol and requires the user's personal or business WhatsApp
account. The user's phone must have WhatsApp installed and an active internet connection
during the initial link.
</Note>

## Option A — In-app QR scan (recommended)

The container generates a QR code; the iOS app renders it in a `WKWebView`. The user scans
it from their WhatsApp app (Settings → Linked Devices → Link a Device).

### User journey

```
User                iOS App              file-api / container
 │                     │                    │
 │  Tap "Add WhatsApp" │                    │
 │────────────────────►│                    │
 │                     │  GET /qr/whatsapp  │
 │                     │───────────────────►│
 │                     │                    │  Start Baileys QR session
 │                     │◄───────────────────│  (SSE stream of QR updates)
 │                     │  Render QR in WKWebView
 │                     │                    │
 │  Open WhatsApp      │                    │
 │  Scan QR code       │                    │
 │────────────────────►│                    │
 │                     │                    │  Baileys receives link confirmation
 │                     │                    │  Write session to container volume
 │                     │  ✓ Connected       │
 │◄────────────────────│                    │
```

### Container-side QR endpoint

The OpenClaw container exposes a lightweight SSE endpoint that the iOS app polls. The file-api
proxies the request with Supabase JWT auth:

```
GET https://files.spark.ooo/qr/whatsapp     ← example; replace with your file-api host
Authorization: Bearer <supabase-jwt>

Response (SSE):
event: qr
data: {"svg": "<svg ...>"}   ← rendered each time QR refreshes (every ~20 s)

event: linked
data: {"phone": "+1555..."}   ← sent once the account is linked

event: error
data: {"message": "QR expired"}
```

### iOS implementation

```swift
import WebKit

class WhatsAppConnectView: UIViewController {

    private var webView: WKWebView!
    private var sseTask: URLSessionDataTask?

    override func viewDidLoad() {
        super.viewDidLoad()
        webView = WKWebView(frame: view.bounds)
        view.addSubview(webView)
        startQRStream()
    }

    func startQRStream() {
        var req = URLRequest(url: URL(string: "\(fileAPIURL)/qr/whatsapp")!)
        req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
        req.setValue("text/event-stream", forHTTPHeaderField: "Accept")

        sseTask = URLSession.shared.dataTask(with: req) { [weak self] data, _, _ in
            guard let data, let text = String(data: data, encoding: .utf8) else { return }
            self?.handleSSEEvent(text)
        }
        sseTask?.resume()
    }

    func handleSSEEvent(_ text: String) {
        // Parse "event: qr\ndata: {...}" lines
        let lines = text.components(separatedBy: "\n")
        var eventType = ""
        var eventData = ""
        for line in lines {
            if line.hasPrefix("event: ") { eventType = String(line.dropFirst(7)) }
            if line.hasPrefix("data: ") { eventData = String(line.dropFirst(6)) }
        }
        DispatchQueue.main.async {
            if eventType == "qr", let json = try? JSONSerialization.jsonObject(with: Data(eventData.utf8)) as? [String: Any],
               let svg = json["svg"] as? String {
                // Render the SVG in WKWebView
                let html = "<html><body style='margin:0;background:#fff'>\(svg)</body></html>"
                self.webView.loadHTMLString(html, baseURL: nil)
            } else if eventType == "linked" {
                self.onLinked(eventData)
            }
        }
    }

    func onLinked(_ jsonData: String) {
        // Save success state, dismiss QR screen, show confirmation
        sseTask?.cancel()
        showConfirmation()
    }
}
```

---

## Option B — Phone number link via deep link

WhatsApp supports a `wa.me` linking scheme that opens the WhatsApp app for cross-device pairing.
This bypasses QR scanning when the user is already on the same device:

```swift
func openWhatsAppLink() {
    // Opens WhatsApp on the same device; user taps "Link this device"
    let url = URL(string: "https://wa.me/qr/\(qrToken)")!
    UIApplication.shared.open(url)
}
```

<Note>
The `wa.me/qr/<token>` deep link requires a QR token generated by Baileys. Retrieve it from
the same SSE endpoint and encode it as the URL path. This flow is faster on device but
requires WhatsApp to be installed on the same iPhone.
</Note>

---

## Session persistence

Once linked, Baileys stores the session credentials (auth state + keys) in the container's
Azure Files volume. The iOS app does not need to re-present the QR code after the initial
link — the session persists across container restarts as long as the user's WhatsApp account
remains connected and active.

If the user logs out of WhatsApp Web (Settings → Linked Devices → remove the OpenClaw entry),
the container detects the disconnect and the iOS app shows a "Reconnect" prompt that restarts
the QR flow.

## Gateway config written to container

```json5
// /data/credentials/whatsapp.json
{
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

The Baileys session files (auth state, signal keys) are stored separately in
`/data/sessions/whatsapp/` and managed by the container automatically.

## Handling QR expiry

QR codes expire after approximately 20 seconds. The container's Baileys instance emits a new
QR event every time a refresh occurs. The iOS app must keep the SSE connection open and render
each new QR until the `linked` event arrives. Show a spinner and instructional copy during
the wait:

> **Waiting for QR scan…**
> In WhatsApp, go to **Settings → Linked Devices → Link a Device** and scan this code.

## Status polling after connect

```swift
func pollWhatsAppStatus() async throws -> Bool {
    var req = URLRequest(url: URL(string: "\(fileAPIURL)/status/channels/whatsapp")!)
    req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
    let (data, _) = try await URLSession.shared.data(for: req)
    let status = try JSONDecoder().decode(ChannelStatus.self, from: data)
    return status.connected
}
```

## Related docs

- [WhatsApp channel](/channels/whatsapp) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/ios-quickconnect)
