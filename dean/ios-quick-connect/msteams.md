---
title: "Microsoft Teams – iOS Quick Connect"
summary: "How to connect a Microsoft Teams bot to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the MS Teams quick-connect screen in the iOS app
---

# Microsoft Teams – iOS Quick Connect

**Auth method:** Guided form (Azure credentials) or OAuth "Sign in with Microsoft"  
**Complexity:** High

Microsoft Teams requires an **Azure Bot registration** before OpenClaw can connect. This involves creating an Azure App Registration and a Bot Service resource. Because this is a multi-step process, the iOS app should guide the user through it with a wizard-style UI.

Two approaches are available:

1. **Guided credential form** — User creates an Azure Bot manually and pastes the App ID, App Password, and Tenant ID.
2. **"Sign in with Microsoft" OAuth** — App handles registration on behalf of the user via Microsoft's OAuth2 flow. This is technically complex and requires a dedicated Azure App registration with delegated permissions.

---

## Option A: Guided credential form (recommended for V1)

### User flow

```
1. User opens Settings → Channels → Microsoft Teams
2. iOS app shows a setup wizard:
   Step 1: "Create an Azure Bot"
         - Button: "Open Azure Portal" (SFSafariViewController → portal.azure.com)
         - Checklist of steps to complete in the portal
   Step 2: Enter credentials
         - Text field: App ID (Application / Client ID)
         - SecureField: App Password (Client Secret)
         - Text field: Tenant ID (Directory / Tenant ID)
   Step 3: Confirm webhook
         - Display: the container's public webhook URL
         - Instruction: "Set this as your bot's messaging endpoint in Azure"
3. User taps "Connect" → credentials are validated then saved
4. iOS app shows success screen with bot App ID
```

### Credential collection UI (SwiftUI sketch)

```swift
struct TeamsConnectView: View {
    @State private var appId = ""
    @State private var appPassword = ""
    @State private var tenantId = ""
    @State private var step = 1
    @State private var status: ConnectionStatus = .idle

    var webhookURL: String {
        // Each container has a unique public endpoint via Cloudflare Tunnel
        ContainerService.shared.containerInfo?.webhookBaseURL.appending("/api/messages") ?? "Loading…"
    }

    var body: some View {
        Form {
            if step == 1 {
                // Step 1: instruction card
                Section {
                    Link("Open Azure Portal", destination: URL(string: "https://portal.azure.com/#create/Microsoft.AzureBot")!)
                    SetupChecklistView(steps: [
                        "Create an Azure Bot resource",
                        "Under Configuration, set Messaging endpoint to your webhook URL (shown in Step 3)",
                        "Under Manage → Certificates & secrets, create a new client secret",
                        "Copy the App ID, client secret, and tenant ID",
                    ])
                    Button("Next") { step = 2 }
                }
            } else if step == 2 {
                // Step 2: credential fields
                Section(header: Text("App ID")) {
                    TextField("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", text: $appId)
                        .autocorrectionDisabled().autocapitalization(.none)
                }
                Section(header: Text("App Password (Client Secret)")) {
                    SecureField("", text: $appPassword).textContentType(.password)
                }
                Section(header: Text("Tenant ID")) {
                    TextField("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", text: $tenantId)
                        .autocorrectionDisabled().autocapitalization(.none)
                }
                Button("Next") { step = 3 }
            } else {
                // Step 3: webhook URL
                Section(header: Text("Messaging Endpoint")) {
                    Text(webhookURL)
                        .font(.footnote.monospaced())
                    Button("Copy") { UIPasteboard.general.string = webhookURL }
                }
                Text("Paste this URL into your Azure Bot's Messaging Endpoint field, then tap Connect.")
                    .font(.footnote).foregroundStyle(.secondary)
                Button("Connect") { Task { await connect() } }
                    .disabled(appId.isEmpty || appPassword.isEmpty)
            }
        }
        .navigationTitle("Connect Microsoft Teams")
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "msteams",
                config: [
                    "enabled": true,
                    "appId": appId,
                    "appPassword": appPassword,
                    "tenantId": tenantId,
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

## Option B: Sign in with Microsoft OAuth (advanced)

The iOS app can use `ASWebAuthenticationSession` to acquire a delegated Microsoft token, then use the Microsoft Graph API to create an Azure Bot on behalf of the user. This is the smoothest UX but requires:

- An Azure App Registration with `AzureBot.ReadWrite.All` and `Application.ReadWrite.All` permissions.
- A backend service to hold the client secret and exchange the authorization code.

```swift
private func signInWithMicrosoft() async {
    let clientId = AppConfig.azureClientId
    let tenant = "common"
    let redirectURI = "https://files.spark.ooo/oauth/msteams/callback"
    let scopes = "https://management.azure.com/.default offline_access"

    var components = URLComponents(string: "https://login.microsoftonline.com/\(tenant)/oauth2/v2.0/authorize")!
    components.queryItems = [
        .init(name: "client_id", value: clientId),
        .init(name: "response_type", value: "code"),
        .init(name: "redirect_uri", value: redirectURI),
        .init(name: "scope", value: scopes),
        .init(name: "state", value: generateStateToken()),
    ]

    let session = ASWebAuthenticationSession(
        url: components.url!,
        callbackURLScheme: "openclaw"
    ) { callbackURL, _ in
        guard let url = callbackURL,
              let code = URLComponents(url: url, resolvingAgainstBaseURL: false)?
                .queryItems?.first(where: { $0.name == "code" })?.value else { return }
        Task { await self.exchangeCode(code) }
    }
    session.prefersEphemeralWebBrowserSession = true
    session.start()
}
```

---

## Container config applied

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<AZURE_APP_ID>",
      appPassword: "<CLIENT_SECRET>",
      tenantId: "<TENANT_ID>",
      webhook: {
        port: 3978,
        path: "/api/messages",
      },
    },
  },
}
```

---

## Webhook exposure

Teams bots require a publicly accessible HTTPS endpoint. Each user's container must expose its `/api/messages` endpoint via:

- **Cloudflare Tunnel** (recommended) — Tunnel URL shown in the iOS app setup wizard.
- **AKS Ingress** — Per-user subdomain (e.g., `<user-id>.gateway.openclaw.ai`).

The iOS app should display the container's webhook URL in the setup wizard so the user can paste it into the Azure Bot configuration.

---

## Security notes

- Azure App Passwords (client secrets) are equivalent to admin credentials; never log or display them.
- For multi-tenant bots, validate the `tenantId` in every inbound message to prevent cross-tenant message injection.
- Restrict the Azure Bot's allowed tenants to the user's specific tenant when possible.
- Rotate client secrets on a schedule; the iOS app settings screen should surface a "Rotate secret" action.
