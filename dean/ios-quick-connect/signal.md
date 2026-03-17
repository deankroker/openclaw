---
title: "Signal – iOS Quick Connect"
summary: "How to connect a Signal number to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Signal quick-connect screen in the iOS app
---

# Signal – iOS Quick Connect

**Auth method:** QR link (recommended) or phone registration  
**Complexity:** High

Signal integration uses `signal-cli` running inside the container. There are two registration paths available from the iOS app:

1. **QR link** — Link the container as a secondary device to an existing Signal account by scanning a QR code. The user does not need a separate phone number.
2. **Phone registration** — Register a dedicated phone number with Signal (requires receiving an SMS or voice verification).

---

## Path A: QR link (recommended for iOS app UX)

This is the simplest path for iOS app users. It links the OpenClaw container as a secondary "linked device" to the user's existing Signal account, similar to Signal Desktop.

### User flow

```
1. User opens Settings → Channels → Signal
2. Taps "Link as Device"
3. iOS app calls the container to start a link session
   → Container runs: signal-cli link -n "OpenClaw"
   → Returns a linking URI (tsdevice://?uuid=...&pub_key=...)
4. iOS app renders the URI as a QR code
5. User opens Signal on their primary phone:
   Settings → Linked Devices → "+" → scan QR
6. Container confirms successful link via polling
7. iOS app shows success with the linked account name
```

### iOS implementation

```swift
struct SignalLinkView: View {
    @State private var linkingQR: UIImage? = nil
    @State private var status: LinkStatus = .idle

    var body: some View {
        VStack(spacing: 24) {
            switch status {
            case .loading:
                ProgressView("Starting link session…")
            case .scanning:
                if let qr = linkingQR {
                    QRCodeView(image: qr)
                    Text("Open Signal → Settings → Linked Devices → tap \"+\"")
                        .font(.footnote)
                        .multilineTextAlignment(.center)
                        .foregroundStyle(.secondary)
                }
            case .linked:
                Label("Signal linked!", systemImage: "checkmark.circle.fill")
                    .foregroundStyle(.green)
            case .error(let msg):
                Label(msg, systemImage: "exclamationmark.triangle")
                    .foregroundStyle(.red)
                Button("Try again") { Task { await startLink() } }
            default:
                Button("Link as Signal Device") { Task { await startLink() } }
            }
        }
        .navigationTitle("Link Signal")
    }

    private func startLink() async {
        status = .loading
        do {
            // Container returns the tsdevice:// URI as a string
            let uri = try await ContainerService.shared.startSignalLink()
            // generateQRCode: use a QR library such as CoreImage's CIFilter("CIQRCodeGenerator")
            // or a package like https://github.com/fwcd/swift-qrcode-generator
            linkingQR = generateQRCode(from: uri)
            status = .scanning
            Task { await pollForLink() }
        } catch {
            status = .error(error.localizedDescription)
        }
    }

    private func pollForLink() async {
        for _ in 0..<40 {
            try? await Task.sleep(nanoseconds: 3_000_000_000)
            if let linked = try? await ContainerService.shared.checkSignalLinkStatus(), linked {
                status = .linked
                return
            }
        }
        status = .error("Link timed out. Please try again.")
    }
}
```

### Container endpoints required

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/gateway/signal/link/start` | POST | Runs `signal-cli link` and returns the `tsdevice://` URI |
| `/gateway/signal/link/status` | GET | Returns `{ "status": "pending" \| "linked" \| "error" }` |

---

## Path B: Phone number registration

For users who want a dedicated Signal bot number (not linked to their personal account).

### User flow

```
1. User opens Settings → Channels → Signal
2. Taps "Register a phone number"
3. Enters the phone number in E.164 format (+15551234567)
4. Selects verification method: SMS or Voice
5. iOS app sends the registration request to the container
6. Container calls Signal's registration API
7. User receives a verification code via SMS or voice call
8. User enters the 6-digit code in the iOS app
9. Container completes registration
10. iOS app shows success
```

### iOS implementation

```swift
struct SignalRegisterView: View {
    @State private var phoneNumber = ""
    @State private var verificationCode = ""
    @State private var step: RegisterStep = .enterPhone
    @State private var captchaToken = ""

    var body: some View {
        Form {
            switch step {
            case .enterPhone:
                Section(header: Text("Phone Number")) {
                    TextField("+15551234567", text: $phoneNumber)
                        .keyboardType(.phonePad)
                        .textContentType(.telephoneNumber)
                }
                Section {
                    Button("Send SMS") { Task { await requestSMS() } }
                    Button("Request Voice Call") { Task { await requestVoice() } }
                }

            case .enterCode:
                Section(header: Text("Verification Code")) {
                    TextField("123456", text: $verificationCode)
                        .keyboardType(.numberPad)
                        .textContentType(.oneTimeCode)
                }
                Section {
                    Button("Verify") { Task { await verifyCode() } }
                }
            }
        }
        .navigationTitle("Register Signal Number")
    }

    private func requestSMS() async {
        try? await ContainerService.shared.signalRegister(
            phoneNumber: phoneNumber,
            method: "sms",
            captcha: captchaToken
        )
        step = .enterCode
    }

    private func verifyCode() async {
        try? await ContainerService.shared.signalVerify(
            phoneNumber: phoneNumber,
            code: verificationCode
        )
    }
}
```

> **Captcha requirement:** Signal's registration API may require a captcha token. The iOS app should present a WebView loading `https://signalcaptchas.org/registration/generate.html` and capture the resulting token before initiating registration.

---

## Container config applied

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",   // registered/linked number
      cliPath: "signal-cli",     // path inside the container
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
}
```

The config is applied after successful registration or linking:

```http
PATCH /config/channels
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{
  "channel": "signal",
  "config": {
    "enabled": true,
    "account": "+15551234567",
    "cliPath": "signal-cli",
    "dmPolicy": "pairing"
  }
}
```

---

## Container requirements

- `signal-cli` must be pre-installed in the OpenClaw container image.
- Java (JRE 17+) or the native binary variant of `signal-cli` must be available.
- The container must be able to reach Signal's servers over HTTPS.

---

## Security notes

- The QR link method is strongly preferred for iOS app users. It avoids the complexity of captcha handling and does not require a separate phone number.
- A linked device can read all messages delivered to the primary account. Advise users to use a secondary Signal account if privacy between the bot and personal messages is important.
- Phone number registration permanently associates the number with a Signal identity. This cannot easily be undone.
- Store `signal-cli` data directory within the user's isolated Azure Files volume so it is not accessible to other users.
