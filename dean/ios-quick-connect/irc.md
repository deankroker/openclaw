---
title: "IRC – iOS Quick Connect"
summary: "How to connect an IRC server to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the IRC quick-connect screen in the iOS app
---

# IRC – iOS Quick Connect

**Auth method:** Guided form (server + nick + optional password)  
**Complexity:** Low

IRC is a classic text-based chat protocol. OpenClaw connects as an IRC client. The user provides a server address, port, nickname, and optional NickServ/SASL credentials. No external developer accounts or OAuth flows are needed.

---

## User flow

```
1. User opens Settings → Channels → IRC
2. iOS app shows a form:
   - Text field: "Server" (e.g., irc.libera.chat)
   - Number field: "Port" (default: 6697)
   - Toggle: "Use TLS" (default: on)
   - Text field: "Nickname"
   - SecureField: "NickServ Password" (optional)
   - Text field: "Channel(s) to join" (comma-separated, e.g., #openclaw,#help)
3. User taps "Connect"
4. Container config is patched; gateway connects to IRC
5. Success screen shows connection status
```

---

## iOS implementation

### 1. UI (SwiftUI sketch)

```swift
struct IRCConnectView: View {
    @State private var server = "irc.libera.chat"
    @State private var port = "6697"
    @State private var useTLS = true
    @State private var nickname = ""
    @State private var password = ""
    @State private var channels = "#openclaw"
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Form {
            Section(header: Text("Server")) {
                TextField("irc.libera.chat", text: $server)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
                    .keyboardType(.URL)
                TextField("Port", text: $port)
                    .keyboardType(.numberPad)
                Toggle("Use TLS", isOn: $useTLS)
            }

            Section(header: Text("Identity")) {
                TextField("Nickname", text: $nickname)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
                SecureField("NickServ Password (optional)", text: $password)
                    .textContentType(.password)
            }

            Section(header: Text("Channels to join (comma-separated)")) {
                TextField("#openclaw,#help", text: $channels)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }

            Section {
                Button("Connect") { Task { await connect() } }
                    .disabled(server.isEmpty || nickname.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect IRC")
    }

    private func connect() async {
        status = .connecting
        let channelList = channels.split(separator: ",").map(String.init).map { $0.trimmingCharacters(in: .whitespaces) }
        var config: [String: Any] = [
            "enabled": true,
            "server": server,
            "port": Int(port) ?? 6697,
            "tls": useTLS,
            "nickname": nickname,
            "channels": channelList,
        ]
        if !password.isEmpty { config["nickservPassword"] = password }
        do {
            try await ChannelConfigService.shared.patch(channel: "irc", config: config)
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
    irc: {
      enabled: true,
      server: "irc.libera.chat",
      port: 6697,
      tls: true,
      nickname: "openclaw-bot",
      nickservPassword: "<NICKSERV_PASSWORD>",   // optional
      channels: ["#openclaw", "#help"],
      dmPolicy: "pairing",
    },
  },
}
```

---

## Popular IRC networks (quick reference)

The iOS app can show a picker of popular IRC networks to simplify server entry:

| Network | Server | TLS Port |
|---------|--------|----------|
| Libera.Chat | `irc.libera.chat` | 6697 |
| OFTC | `irc.oftc.net` | 6697 |
| freenode | `irc.freenode.net` | 7000 |
| EFnet | `irc.efnet.org` | 6697 |
| IRCnet | `open.ircnet.net` | 6697 |

---

## Security notes

- Always use TLS (port 6697 or 7000 depending on the network) to protect credentials in transit.
- NickServ passwords are stored on the container and used only during registration. Do not use the same password as other accounts.
- IRC is a low-security protocol; do not use it to exchange sensitive information.
- Consider using SASL authentication instead of NickServ if the IRC network supports it.
