---
summary: "Quick connect iMessage from the managed iOS app using BlueBubbles server credentials"
read_when:
  - Implementing iMessage channel setup in the iOS app
  - Understanding BlueBubbles server URL and password flow
title: "iMessage — iOS Quick Connect"
sidebarTitle: "iMessage (BlueBubbles)"
---

# iMessage — iOS Quick Connect

**Pattern: BlueBubbles server URL + password**

iMessage requires a Mac running [BlueBubbles](https://bluebubbles.app) — a macOS server
application that interfaces with the Messages app. The iOS OpenClaw app collects the
BlueBubbles server URL and API password, then writes them to the container so the Gateway
can communicate with the user's Mac.

<Note>
iMessage is **not available as a fully managed/cloud-only integration**. The user must have
a Mac (or macOS VM) running BlueBubbles with an Apple ID signed in to Messages. The
OpenClaw container acts as the AI brain, but it must be able to reach the user's BlueBubbles
server over the internet.
</Note>

## Architecture

```
iOS App  ──►  OpenClaw Container (Azure)  ──►  BlueBubbles Server (user's Mac)
                                                      │
                                              Messages.app / iMessage
```

The container's Gateway uses the BlueBubbles REST API + WebSocket to send and receive iMessages.
The BlueBubbles server must be reachable from Azure — either via a port-forwarded public IP,
Cloudflare Tunnel, Tailscale, or a similar tunneling solution.

## What the iOS app collects

| Field | UI label | Required | Notes |
| --- | --- | --- | --- |
| `serverUrl` | BlueBubbles server URL | Yes | Example: `https://bb.yourname.tailnet.ts.net` |
| `password` | Server password | Yes | Set in BlueBubbles → Settings → API |

## User journey

```
User                iOS App              file-api               OpenClaw Pod
 │                     │                    │                        │
 │  Tap "Add iMessage" │                    │                        │
 │────────────────────►│                    │                        │
 │                     │  Show server URL   │                        │
 │                     │  + password fields │                        │
 │◄────────────────────│                    │                        │
 │  Enter URL + pass   │                    │                        │
 │────────────────────►│                    │                        │
 │                     │  Validate against  │                        │
 │                     │  BlueBubbles /api/v1/ping                   │
 │                     │───────────────────────────────────────────  │
 │                     │  POST /credentials/bluebubbles              │
 │                     │────────────────────►                        │
 │                     │                    │───────────────────────►│
 │                     │                    │                        │  Reload config
 │                     │  ✓ Connected       │                        │◄──────────────
 │◄────────────────────│                    │                        │
```

## Implementation guide

### Step 1 — Validate the BlueBubbles server

Before saving, ping the BlueBubbles server to verify the URL and password are correct:

```swift
struct BlueBubblesCredential: Codable {
    let serverUrl: String
    let password: String
}

func validateBlueBubbles(_ cred: BlueBubblesCredential) async throws -> BlueBubblesPingResponse {
    // BlueBubbles REST API: GET /api/v1/ping?password=<password>
    var components = URLComponents(string: "\(cred.serverUrl)/api/v1/ping")!
    components.queryItems = [URLQueryItem(name: "password", value: cred.password)]

    let (data, response) = try await URLSession.shared.data(from: components.url!)
    guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
        throw ConnectError.unreachable
    }
    return try JSONDecoder().decode(BlueBubblesPingResponse.self, from: data)
}
// BlueBubblesPingResponse: { status: 200, message: "pong", data: { url: String, private_api: Bool } }
```

### Step 2 — Write credentials to the container

```swift
func saveBlueBubblesCredential(_ cred: BlueBubblesCredential) async throws {
    var req = URLRequest(url: URL(string: "\(fileAPIURL)/credentials/bluebubbles")!)
    req.httpMethod = "POST"
    req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
    req.setValue("application/json", forHTTPHeaderField: "Content-Type")
    req.httpBody = try JSONEncoder().encode(cred)
    let (_, resp) = try await URLSession.shared.data(for: req)
    guard (resp as? HTTPURLResponse)?.statusCode == 200 else {
        throw ConnectError.saveFailed
    }
}
```

### Step 3 — Reload the Gateway

```swift
try await reloadChannel("bluebubbles")
```

### Step 4 — Confirm connection

After reload, poll `GET /status/channels/bluebubbles` until `connected == true`. Show the user
the Apple ID / phone number associated with the BlueBubbles server (available from the ping
response).

## BlueBubbles server setup guide (in-app)

Since this requires the user to set up their own Mac server, include an in-app guide:

1. **Install BlueBubbles** — [bluebubbles.app](https://bluebubbles.app) on macOS 12+.
2. **Sign in to Messages** — The same Apple ID that has iMessage enabled.
3. **Configure BlueBubbles** — Under Settings → Server, enable the HTTP server and set a
   strong password. Note the server URL (or set up Cloudflare Tunnel / Tailscale for remote access).
4. **Enable Private API** (optional) — Enables typing indicators, reactions, and editing.
   Requires Private API helper installation.
5. **Test connectivity** — Use `curl` or BlueBubbles' built-in API tester to verify
   `https://your-server/api/v1/ping?password=<password>` returns `200 OK`.

## Server accessibility options

| Method | Complexity | Notes |
| --- | --- | --- |
| Public IP + port | Low | Requires router port-forward; not recommended for home networks |
| Cloudflare Tunnel | Medium | Free; installs `cloudflared` on Mac; stable and secure |
| Tailscale | Low | Best for personal use; requires Tailscale on both Mac and container |
| ngrok | Low | Free tier limited; useful for testing only |

Include a "Test Connection" button in the iOS app that retries the ping endpoint so users can
troubleshoot connectivity issues before saving.

## Gateway config written to container

```json5
// /data/credentials/bluebubbles.json
{
  channels: {
    bluebubbles: {
      enabled: true,
      url: "https://bb.yourname.tailnet.ts.net",
      password: "super-secret-password",
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
    },
  },
}
```

## Limitations

- **macOS required**: iMessage cannot run without Apple hardware running macOS.
- **Apple ID session**: If the user's Mac restarts or the Apple ID session expires, the
  connection drops until the user re-signs in to Messages on their Mac.
- **macOS 26 Tahoe note**: Message editing is currently broken in BlueBubbles on macOS 26.
  Other features (send, receive, reactions, unsend) remain functional.
- **Not cloud-native**: This integration is inherently hybrid — the AI runs in the cloud but
  iMessage delivery still requires the user's Mac to be online.

## Related docs

- [BlueBubbles channel](/channels/bluebubbles) — full channel reference
- [iMessage (legacy)](/channels/imessage) — deprecated legacy integration
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/ios-quickconnect)
