---
title: "Telegram – iOS Quick Connect"
summary: "How to connect a Telegram bot to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Telegram quick-connect screen in the iOS app
---

# Telegram – iOS Quick Connect

**Auth method:** Token paste  
**Complexity:** Low

Telegram bot tokens are obtained from **@BotFather** inside Telegram itself. The user creates a bot in under a minute, copies the token, and pastes it into the iOS app. No browser-based OAuth or QR scan is needed.

---

## User flow

```
1. User opens Settings → Channels → Telegram
2. iOS app shows a "Connect Telegram" screen with:
   - "Open BotFather in Telegram" button (deep-links to tg://resolve?domain=BotFather)
   - Instruction card: "Send /newbot, follow prompts, copy the token"
   - Secure text field for "Bot Token"
3. User taps "Connect" → token is validated, then saved to the container
4. Success screen shows bot @handle and link to start a conversation
```

---

## iOS implementation

### 1. Deep-link to BotFather

```swift
let botFatherURL = URL(string: "tg://resolve?domain=BotFather")!
if UIApplication.shared.canOpenURL(botFatherURL) {
    await UIApplication.shared.open(botFatherURL)
} else {
    // Telegram not installed: open web fallback
    await UIApplication.shared.open(URL(string: "https://t.me/BotFather")!)
}
```

### 2. UI (SwiftUI sketch)

```swift
struct TelegramConnectView: View {
    @State private var botToken = ""
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Form {
            Section {
                Button("Open BotFather in Telegram") {
                    openBotFather()
                }
                .foregroundStyle(.blue)
            } header: {
                Text("Step 1 – Create a bot")
            } footer: {
                Text("Send /newbot, choose a name and username, then copy the token BotFather gives you.")
            }

            Section(header: Text("Step 2 – Paste your token")) {
                SecureField("123456789:ABCDEFabcdef...", text: $botToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
            }

            Section {
                Button("Connect") {
                    Task { await connect() }
                }
                .disabled(botToken.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect Telegram")
    }
}
```

### 3. Token validation (optional pre-check)

```swift
func validateTelegramToken(_ token: String) async throws -> String {
    let url = URL(string: "https://api.telegram.org/bot\(token)/getMe")!
    let (data, response) = try await URLSession.shared.data(from: url)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw ChannelError.invalidToken
    }
    struct TelegramUser: Decodable { struct Result: Decodable { let username: String? } let result: Result }
    let me = try JSONDecoder().decode(TelegramUser.self, from: data)
    return me.result.username ?? "bot"
}
```

---

## Container config applied

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "<BOT_TOKEN>",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

Sent via the file-api config-patch endpoint:

```http
PATCH /config/channels
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{
  "channel": "telegram",
  "config": {
    "enabled": true,
    "botToken": "<BOT_TOKEN>",
    "dmPolicy": "pairing"
  }
}
```

---

## Approving the first pairing request

After the container connects to Telegram, the first user to DM the bot will need to be approved (because `dmPolicy` defaults to `pairing`). The iOS app can expose this through a **Pending Approvals** screen that polls the container's pairing list endpoint:

```
GET /gateway/pairing?channel=telegram
→ [{ "code": "XXXX", "from": "@alice", "requestedAt": "..." }]
```

The user taps **Approve** in the app, which calls:

```
POST /gateway/pairing/approve
{ "channel": "telegram", "code": "XXXX" }
```

Alternatively, set `dmPolicy: "open"` to skip pairing for trusted single-user deployments.

---

## Security notes

- The bot token grants full control of the bot; treat it like a password.
- Consider setting `allowFrom` to the user's own Telegram user ID to prevent unauthorized access to the bot.
- Tokens should be stored only on the container side; do not persist them in the iOS Keychain beyond the initial setup POST.
