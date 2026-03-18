---
summary: "How to connect each chat provider quickly from the OpenClaw managed iOS app"
read_when:
  - Building the in-app channel connect flow
  - Deciding which auth pattern to implement for a provider
title: "iOS Quick Connect"
sidebarTitle: "Overview"
---

# iOS Quick Connect

This section describes how to implement a **quick connect** flow for each supported chat provider,
assuming users are running OpenClaw through the managed iOS app backed by Supabase authentication
and per-user Azure Kubernetes containers.

## Architecture recap

```
┌─────────────────┐    ┌──────────────┐    ┌──────────────────────────┐
│                 │    │              │    │  Azure Cloud (AKS)       │
│   iOS App       │◄──►│  Supabase    │    │                          │
│  (SwiftUI)      │    │  (Auth + DB) │    │  ┌────────────────────┐  │
│                 │    │              │    │  │  OpenClaw Pod       │  │
│ • Supabase JWT  │    │ • Users      │    │  │  (ClusterIP only)   │  │
│ • Channel setup │    │ • Containers │    │  └─────────┬──────────┘  │
│ • Token storage │    │ • RLS        │    │            │             │
│                 │    │              │    │  ┌─────────┴──────────┐  │
│                 │    │              │    │  │  file-api pod       │  │
│                 │    │              │    │  │  (Cloudflare Tunnel)│  │
└────────┬────────┘    └──────────────┘    │  └────────────────────┘  │
         │                                 └──────────────────────────┘
         │  HTTPS + Supabase JWT
         └──▶ files.spark.ooo
```

Every OpenClaw container runs a full Gateway process. Channel credentials (bot tokens, session
files, OAuth tokens) are stored in the container's Azure Files volume and never touch the iOS
app's local storage. The iOS app collects the minimum information needed to bootstrap a
connection, then sends it to the container over the secure file-api endpoint.

## Connect patterns at a glance

| Provider | Best pattern | Complexity | In-app OAuth? |
| --- | --- | --- | --- |
| [Telegram](/dean/ios-quickconnect/telegram) | Paste bot token | Low | No |
| [Discord](/dean/ios-quickconnect/discord) | Paste bot token + guild ID | Low | Optional |
| [WhatsApp](/dean/ios-quickconnect/whatsapp) | QR code scan via WebView | Medium | No |
| [Slack](/dean/ios-quickconnect/slack) | OAuth install flow (ASWebAuthenticationSession) | Medium | Yes |
| [Signal](/dean/ios-quickconnect/signal) | Phone number + SMS code | Medium | No |
| [iMessage (BlueBubbles)](/dean/ios-quickconnect/imessage) | BlueBubbles server URL + password | Medium | No |
| [Matrix](/dean/ios-quickconnect/matrix) | Access token or password login | Low–Medium | Optional |
| [Microsoft Teams](/dean/ios-quickconnect/msteams) | Azure AD OAuth (enterprise SSO) | High | Yes |
| [Google Chat](/dean/ios-quickconnect/googlechat) | Service account JSON or OAuth | High | Yes |

## Shared infrastructure

### Credential delivery

1. User provides credentials in the iOS app (paste, QR scan, or OAuth callback).
2. App `POST`s the credential payload to `https://files.spark.ooo/credentials/<channel>` using
   the user's Supabase JWT for authentication.
3. The file-api pod writes the credential to the Azure Files volume at a path that the user's
   OpenClaw container can read (for example, `/data/credentials/telegram.json`).
4. The iOS app calls a lightweight RPC endpoint on the container (via the same Cloudflare Tunnel)
   to tell the Gateway to reload the channel config.

### Token storage conventions

Credentials are written to the container's `/data/credentials/<channel>.json`. The file-api
enforces per-user path isolation via the Supabase RLS policy — a user can only write to their
own container's volume.

### OAuth redirect handling

For providers that require OAuth, the iOS app uses `ASWebAuthenticationSession` (or
`SFSafariViewController` for flows that need cookie sharing). The redirect URI is a universal
link registered on `https://openclaw.ai/auth/callback/<channel>`, which the iOS app handles
via `UIApplicationDelegate` → `openURL`.

## Related docs

- [iOS App (node)](/platforms/ios)
- [Gateway configuration](/gateway/configuration)
- [Pairing](/channels/pairing)
- [Channel troubleshooting](/channels/troubleshooting)
