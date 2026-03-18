---
summary: "Quick connect Signal from the managed iOS app using phone number registration or QR link"
read_when:
  - Implementing Signal channel setup in the iOS app
  - Understanding phone registration and QR link flows for Signal
title: "Signal — iOS Quick Connect"
sidebarTitle: "Signal"
---

# Signal — iOS Quick Connect

**Pattern: phone number + SMS/voice code, or QR-based device link**

Signal's `signal-cli` integration requires a phone number that acts as the bot's Signal
identity. The app can either register a new number (useful for dedicated bot numbers) or link
an existing Signal account as an additional linked device.

<Note>
Signal does not have an official bot API. OpenClaw uses `signal-cli` to communicate with the
Signal network. A real phone number is required — VoIP numbers from services like Google Voice
may work but are not guaranteed. Twilio or similar SMS providers are commonly used for
dedicated bot numbers.
</Note>

## Option A — Phone number registration (dedicated bot number)

Best for managed deployments where you provision a dedicated Signal bot number for each user.
The user may not even need to know the underlying number.

### User journey

```
User                iOS App              file-api / container     Signal Network
 │                     │                    │                          │
 │  Tap "Add Signal"   │                    │                          │
 │────────────────────►│                    │                          │
 │                     │  POST /signal/register { phone }              │
 │                     │───────────────────►│                          │
 │                     │                    │  signal-cli register     │
 │                     │                    │──────────────────────────►
 │                     │                    │                          │  SMS/Voice code
 │                     │                    │◄─────────────────────────│
 │  Enter SMS code     │                    │                          │
 │◄────────────────────│                    │                          │
 │  123456             │                    │                          │
 │────────────────────►│                    │                          │
 │                     │  POST /signal/verify { phone, code }          │
 │                     │───────────────────►│                          │
 │                     │                    │  signal-cli verify       │
 │                     │                    │──────────────────────────►
 │                     │                    │◄─────────────────────────│
 │                     │  ✓ Connected       │                          │
 │◄────────────────────│                    │                          │
```

### iOS implementation

```swift
struct SignalRegisterRequest: Encodable {
    let phone: String           // E.164 format: "+15551234567"
    let captcha: String?        // Required by Signal for new registrations
}

struct SignalVerifyRequest: Encodable {
    let phone: String
    let code: String            // 6-digit SMS code (no dashes)
}

class SignalConnectViewModel: ObservableObject {

    // Step 1: Start registration (sends SMS/voice code to the phone)
    func register(phone: String, captcha: String?) async throws {
        var req = URLRequest(url: URL(string: "\(fileAPIURL)/signal/register")!)
        req.httpMethod = "POST"
        req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
        req.setValue("application/json", forHTTPHeaderField: "Content-Type")
        req.httpBody = try JSONEncoder().encode(SignalRegisterRequest(phone: phone, captcha: captcha))
        let (_, resp) = try await URLSession.shared.data(for: req)
        guard (resp as? HTTPURLResponse)?.statusCode == 200 else {
            throw ConnectError.registrationFailed
        }
    }

    // Step 2: Submit the verification code from SMS/voice
    func verify(phone: String, code: String) async throws {
        var req = URLRequest(url: URL(string: "\(fileAPIURL)/signal/verify")!)
        req.httpMethod = "POST"
        req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
        req.setValue("application/json", forHTTPHeaderField: "Content-Type")
        req.httpBody = try JSONEncoder().encode(SignalVerifyRequest(phone: phone, code: code))
        let (_, resp) = try await URLSession.shared.data(for: req)
        guard (resp as? HTTPURLResponse)?.statusCode == 200 else {
            throw ConnectError.verificationFailed
        }
    }
}
```

### Captcha handling

Signal requires a captcha for new number registrations to prevent spam. The iOS app should
load the Signal captcha challenge in a `WKWebView` pointed at `https://signalcaptchas.org/`:

```swift
func loadSignalCaptcha(completion: @escaping (String) -> Void) {
    let webView = WKWebView()
    // Set up script message handler to receive the token
    webView.configuration.userContentController.add(
        CaptchaHandler(completion: completion),
        name: "signalCaptcha"
    )
    webView.load(URLRequest(url: URL(string: "https://signalcaptchas.org/challenge/generate.html")!))
}
```

---

## Option B — QR-based device link (link to existing Signal account)

The user links the bot to their existing Signal account (similar to Signal Desktop). The
container generates a device link URL, which is displayed as a QR code in the app. The user
scans it from Signal on their phone (Settings → Linked Devices → Link New Device).

### User journey

```
User                iOS App              container
 │                     │                    │
 │  Choose "Link       │                    │
 │  existing account"  │                    │
 │────────────────────►│                    │
 │                     │  GET /signal/qr-link
 │                     │───────────────────►│
 │                     │                    │  signal-cli link
 │                     │◄───────────────────│  { qrCodeSVG, linkURL }
 │                     │  Render QR         │
 │                     │                    │
 │  Open Signal app    │                    │
 │  Scan QR code       │                    │
 │────────────────────►│                    │
 │                     │                    │  Link confirmed
 │                     │  ✓ Linked          │◄──────────────
 │◄────────────────────│                    │
```

### Container endpoint

```
GET /signal/qr-link (SSE)
Authorization: Bearer <supabase-jwt>

event: qr
data: {"svg": "<svg ...>", "uri": "sgnl://linkdevice?uuid=...&pub_key=..."}

event: linked
data: {"account": "+15551234567"}
```

```swift
// Render the QR link the same way as the WhatsApp QR flow
// See WhatsApp quick connect for the SSE rendering pattern
```

---

## Managed number provisioning

For fully managed deployments, provision dedicated Signal numbers using an SMS API (for
example Twilio or Vonage) and register them automatically when a new user subscribes:

```typescript
// Pseudocode — runs in your backend at user sign-up
async function provisionSignalForUser(userId: string) {
  const phone = await smsProvider.purchaseNumber("+1"); // Buy a US number
  await containerRPC.exec(userId, ["signal-cli", "register", "-n", phone, "--captcha", await solveSignalCaptcha()]);
  // Note: solveSignalCaptcha() is a placeholder for your captcha-solving implementation.
  // See Option A above and the Signal captcha WebView section for guidance.
  const code = await smsProvider.waitForCode(phone);
  await containerRPC.exec(userId, ["signal-cli", "verify", "-n", phone, code]);
  await fileApi.writeCredential(userId, "signal", {
    account: phone,
    dmPolicy: "pairing",
    allowFrom: [],
  });
  await containerRPC.reloadChannel(userId, "signal");
}
```

This way users never need to provide a phone number — the bot number is pre-provisioned and
the iOS app simply shows "Signal connected" after onboarding.

---

## Gateway config written to container

```json5
// /data/credentials/signal.json
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
}
```

The signal-cli data directory (keys, session) lives at the container's default `signal-cli`
data path (`/root/.local/share/signal-cli`), which is mounted on the Azure Files volume.

## Choosing between Option A and Option B

| | Option A (register number) | Option B (link device) |
| --- | --- | --- |
| User effort | Medium (SMS code entry) | Low (QR scan in Signal) |
| Dedicated number needed | Yes | No (uses user's number) |
| User shares their number | No | Yes (bot linked to their account) |
| Best for | Managed apps, privacy-forward | Developer use, personal agents |

## Related docs

- [Signal channel](/channels/signal) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/dean/ios-quickconnect)
