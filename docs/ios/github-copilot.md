---
summary: "Implement GitHub Copilot quick connect in the OpenClaw iOS app (GitHub OAuth, device flow)"
read_when:
  - Adding GitHub Copilot authentication to the iOS app
  - Implementing GitHub OAuth device flow on iOS
title: "GitHub Copilot — iOS Quick Connect"
---

# GitHub Copilot — iOS Quick Connect

GitHub Copilot uses **GitHub OAuth** for authentication — there is no standalone API key.
Users sign in with their GitHub account, and OpenClaw exchanges the token for a Copilot
API token at runtime. The GitHub mobile app supports OAuth delegation, making this provider
a good candidate for native app deep linking.

## Option A: GitHub Device Flow (no browser required)

The GitHub device flow shows the user a short code and a URL
(`https://github.com/login/device`). They enter the code on any device/browser. The app
polls until authorization completes.

### iOS implementation

```swift
struct GitHubDeviceFlowView: View {
    @State private var userCode: String = ""
    @State private var verificationURL: URL?
    @State private var isPolling = false
    @EnvironmentObject var session: UserSession

    private let clientID = "<your-github-oauth-app-client-id>"

    var body: some View {
        VStack(spacing: 20) {
            if userCode.isEmpty {
                Button("Sign in with GitHub") { Task { await startDeviceFlow() } }
            } else {
                Text("Visit: \(verificationURL?.absoluteString ?? "")")
                    .font(.subheadline)
                Text("Enter code:")
                    .font(.caption)
                Text(userCode)
                    .font(.system(.largeTitle, design: .monospaced))
                    .bold()
                Button("Copy code & open GitHub") {
                    UIPasteboard.general.string = userCode
                    if let url = verificationURL { UIApplication.shared.open(url) }
                }
                if isPolling {
                    ProgressView("Waiting for authorization…")
                }
            }
        }
        .padding()
    }

    private func startDeviceFlow() async {
        // 1. Request device code
        var request = URLRequest(url: URL(string: "https://github.com/login/device/code")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.httpBody = try? JSONSerialization.data(withJSONObject: [
            "client_id": clientID,
            "scope": "user",
        ])

        guard
            let (data, _) = try? await URLSession.shared.data(for: request),
            let response = try? JSONDecoder().decode(DeviceCodeResponse.self, from: data)
        else { return }

        userCode = response.userCode
        verificationURL = URL(string: response.verificationURI)
        isPolling = true

        // 2. Poll for token
        await pollForToken(deviceCode: response.deviceCode, interval: response.interval)
    }

    private func pollForToken(deviceCode: String, interval: Int) async {
        repeat {
            try? await Task.sleep(for: .seconds(interval))
            var request = URLRequest(url: URL(string: "https://github.com/login/oauth/access_token")!)
            request.httpMethod = "POST"
            request.setValue("application/json", forHTTPHeaderField: "Accept")
            request.httpBody = try? JSONSerialization.data(withJSONObject: [
                "client_id": clientID,
                "device_code": deviceCode,
                "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
            ])

            guard
                let (data, _) = try? await URLSession.shared.data(for: request),
                let response = try? JSONDecoder().decode(AccessTokenResponse.self, from: data),
                let token = response.accessToken
            else { continue }

            isPolling = false
            await session.saveProviderKey(provider: "github-copilot", key: token)
            return
        } while isPolling
    }
}

struct DeviceCodeResponse: Decodable {
    let deviceCode: String
    let userCode: String
    let verificationURI: String
    let interval: Int
    enum CodingKeys: String, CodingKey {
        case deviceCode = "device_code"
        case userCode = "user_code"
        case verificationURI = "verification_uri"
        case interval
    }
}

struct AccessTokenResponse: Decodable {
    let accessToken: String?
    enum CodingKeys: String, CodingKey { case accessToken = "access_token" }
}
```

**Why device flow over browser OAuth?** The device flow avoids `ASWebAuthenticationSession`'s
redirect handling complexity and works even when the user is on a slow network, because the
short code entry can be done on any device. It also fits the iOS modal UX better.

---

## Option B: GitHub OAuth PKCE browser flow

For a more traditional in-app OAuth experience, use `ASWebAuthenticationSession` with the
GitHub authorization endpoint.

```swift
var components = URLComponents(string: "https://github.com/login/oauth/authorize")!
components.queryItems = [
    .init(name: "client_id",    value: clientID),
    .init(name: "scope",        value: "user"),
    .init(name: "redirect_uri", value: "https://app.openclaw.ai/oauth/github/callback"),
    .init(name: "state",        value: UUID().uuidString),
]

let session = ASWebAuthenticationSession(
    url: components.url!,
    callbackURLScheme: "https"
) { callbackURL, error in
    // Exchange code via file-api relay
}
session.presentationContextProvider = self
session.start()
```

GitHub's `/login/oauth/authorize` does not require PKCE for server-side OAuth apps, but you
should still exchange the code server-side to keep `client_secret` out of the binary.

---

## Option C: GitHub iOS app deep link

The **GitHub iOS app** (`github://`) supports opening with OAuth context but does not expose
a documented token delegation scheme for third-party apps.

```swift
let githubURL = URL(string: "github://")!
if UIApplication.shared.canOpenURL(githubURL) {
    UIApplication.shared.open(githubURL)
} else {
    UIApplication.shared.open(
        URL(string: "https://apps.apple.com/app/github/id1477376905")!
    )
}
```

Add `github` to `LSApplicationQueriesSchemes` in `Info.plist`.

---

## Option D: OpenClaw URL scheme quick auth

For users who ran `openclaw models auth login-github-copilot` on their desktop gateway:

```
https://app.openclaw.ai/connect?provider=github-copilot&token=<base64-encoded-token>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "github-copilot",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let token = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "github-copilot", key: token) }
}
```

---

## Credential flow to the container

```
User completes GitHub auth (device flow or browser OAuth)
        ↓
iOS app receives GitHub access token
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "github-copilot", "githubToken": "ghp_..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod exchanges GitHub token → Copilot API token at runtime
```

The OpenClaw container (using the `github-copilot` provider) handles the GitHub token →
Copilot token exchange automatically on each session start.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Device flow hangs at polling | User did not complete code entry | Prompt user to visit verification URL |
| `Copilot not enabled` error | GitHub account does not have Copilot | Activate Copilot at github.com/settings/copilot |
| `bad_verification_code` | Device code expired (15 min TTL) | Restart the device flow |
| Model not available | Plan does not include requested model | Try `github-copilot/gpt-4o` or `github-copilot/gpt-4.1` |

## Related docs

- [GitHub Copilot provider (OpenClaw)](/providers/github-copilot)
- [iOS Quick Connect overview](/ios)
