---
summary: "Guide to implementing model switching in a managed OpenClaw iOS app"
read_when:
  - You are building an iOS app that manages OpenClaw containers on behalf of users
  - You need to implement a model picker or model switching UI in native iOS
  - You want to set the primary model, fallbacks, or allowlist from an iOS interface
title: "iOS Managed App — Model Switching"
---

# iOS Managed App — Model Switching

This guide covers how to implement **model switching** in an iOS app that manages dedicated OpenClaw containers. Users select their active model from a native iOS UI; the iOS app writes the change to `openclaw.json` via the file-API bridge (authenticated with the user's Supabase JWT).

For provider authentication (connecting a provider for the first time), see [`ios-managed-app.md`](./ios-managed-app.md).

## Architecture recap

Model configuration lives in `openclaw.json` in the user's mounted Azure Files share. The iOS app reads and writes this file through the file-API pod:

```
iOS app → HTTPS + Supabase JWT → files.spark.ooo → openclaw.json → OpenClaw pod reload
```

## Model config keys

| Key | Purpose |
|---|---|
| `agents.defaults.model.primary` | Active model for all new sessions |
| `agents.defaults.model.fallbacks` | Ordered list tried when primary fails |
| `agents.defaults.models` | Allowlist + aliases (empty = all models allowed) |
| `agents.defaults.imageModel.primary` | Model used when primary can't accept images |
| `agents.defaults.imageGenerationModel.primary` | Model used for image generation |

## Core pattern

Read → patch primary → write → reload:

```swift
// Conceptual — adapt to your actual file-api client
func setActiveModel(providerModel: String) async throws {
    var config = try await fileAPI.readOpenClawConfig()
    config.agents.defaults.model.primary = providerModel
    try await fileAPI.writeOpenClawConfig(config)
    try await fileAPI.reloadGateway()
}
```

## Building the model picker

### 1. Define your catalog

Keep the model catalog in the iOS app so the picker works without a round-trip:

```swift
struct ModelEntry: Identifiable, Hashable {
    let id: String          // "provider/model-id" — used in config
    let displayName: String
    let provider: String
    let tier: ModelTier
}

enum ModelTier: String, CaseIterable {
    case frontier = "Frontier"
    case balanced = "Balanced"
    case fast = "Fast"
    case local = "Local"
}

let modelCatalog: [ModelEntry] = [
    // Frontier
    ModelEntry(id: "anthropic/claude-opus-4-6",     displayName: "Claude Opus 4.6",     provider: "Anthropic",   tier: .frontier),
    ModelEntry(id: "openai/gpt-5.4",                displayName: "GPT-5.4",              provider: "OpenAI",      tier: .frontier),
    ModelEntry(id: "openai/o3",                     displayName: "o3",                   provider: "OpenAI",      tier: .frontier),
    ModelEntry(id: "openai-codex/gpt-5.4",          displayName: "Codex GPT-5.4",        provider: "OpenAI Codex",tier: .frontier),
    ModelEntry(id: "github-copilot/gpt-4.1",        displayName: "Copilot GPT-4.1",      provider: "GitHub Copilot", tier: .frontier),

    // Balanced
    ModelEntry(id: "anthropic/claude-sonnet-4-5",   displayName: "Claude Sonnet 4.5",   provider: "Anthropic",   tier: .balanced),
    ModelEntry(id: "openai/gpt-4.1",                displayName: "GPT-4.1",              provider: "OpenAI",      tier: .balanced),
    ModelEntry(id: "mistral/mistral-large-latest",  displayName: "Mistral Large",        provider: "Mistral",     tier: .balanced),
    ModelEntry(id: "together/moonshotai/Kimi-K2.5", displayName: "Kimi K2.5",            provider: "Together AI", tier: .balanced),
    ModelEntry(id: "openrouter/anthropic/claude-sonnet-4-5", displayName: "Claude Sonnet (OpenRouter)", provider: "OpenRouter", tier: .balanced),

    // Fast
    ModelEntry(id: "anthropic/claude-haiku-4-5",    displayName: "Claude Haiku 4.5",    provider: "Anthropic",   tier: .fast),
    ModelEntry(id: "openai/gpt-5-mini",             displayName: "GPT-5 mini",           provider: "OpenAI",      tier: .fast),
    ModelEntry(id: "mistral/mistral-small-latest",  displayName: "Mistral Small",        provider: "Mistral",     tier: .fast),
    ModelEntry(id: "moonshot/moonshot-v1-8k",       displayName: "Moonshot v1-8k",       provider: "Moonshot",    tier: .fast),

    // Local
    ModelEntry(id: "ollama/llama3.2",               displayName: "Llama 3.2 (Ollama)",  provider: "Ollama",      tier: .local),
    ModelEntry(id: "ollama/qwen2.5-coder",          displayName: "Qwen 2.5 Coder",      provider: "Ollama",      tier: .local),
]
```

### 2. Model picker view

```swift
struct ModelPickerView: View {
    @State private var currentModel: String = ""
    @State private var isLoading = false
    @State private var errorMessage: String?
    let fileAPI: FileAPIClient

    var groupedModels: [(ModelTier, [ModelEntry])] {
        ModelTier.allCases.compactMap { tier in
            let entries = modelCatalog.filter { $0.tier == tier }
            return entries.isEmpty ? nil : (tier, entries)
        }
    }

    var body: some View {
        NavigationStack {
            List {
                ForEach(groupedModels, id: \.0) { tier, entries in
                    Section(tier.rawValue) {
                        ForEach(entries) { entry in
                            ModelRow(
                                entry: entry,
                                isSelected: currentModel == entry.id,
                                onSelect: { selectModel(entry) }
                            )
                        }
                    }
                }
            }
            .navigationTitle("Active Model")
            .overlay {
                if isLoading { ProgressView() }
            }
            .alert("Error", isPresented: .constant(errorMessage != nil)) {
                Button("OK") { errorMessage = nil }
            } message: {
                Text(errorMessage ?? "")
            }
        }
        .task { await loadCurrentModel() }
    }

    private func loadCurrentModel() async {
        do {
            let config = try await fileAPI.readOpenClawConfig()
            currentModel = config.agents?.defaults?.model?.primary ?? ""
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    private func selectModel(_ entry: ModelEntry) {
        isLoading = true
        Task {
            do {
                try await setActiveModel(providerModel: entry.id)
                currentModel = entry.id
            } catch {
                errorMessage = error.localizedDescription
            }
            isLoading = false
        }
    }
}

struct ModelRow: View {
    let entry: ModelEntry
    let isSelected: Bool
    let onSelect: () -> Void

    var body: some View {
        Button(action: onSelect) {
            HStack {
                VStack(alignment: .leading, spacing: 2) {
                    Text(entry.displayName)
                        .foregroundStyle(.primary)
                    Text(entry.provider)
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
                Spacer()
                if isSelected {
                    Image(systemName: "checkmark.circle.fill")
                        .foregroundStyle(.tint)
                }
            }
        }
        .contentShape(Rectangle())
    }
}
```

### 3. Setting fallbacks

For resilience, let users configure an ordered fallback list:

```swift
func setFallbacks(models: [String]) async throws {
    var config = try await fileAPI.readOpenClawConfig()
    config.agents.defaults.model.fallbacks = models
    try await fileAPI.writeOpenClawConfig(config)
    try await fileAPI.reloadGateway()
}

// Example: set Sonnet as fallback behind Opus
try await setFallbacks(models: [
    "anthropic/claude-sonnet-4-5",
    "openai/gpt-4.1",
])
```

### 4. Restricting the allowlist

To limit which models users can pick in-session (via `/model` in chat), set `agents.defaults.models`:

```swift
struct ModelAllowEntry: Codable {
    var alias: String?
}

func setAllowlist(_ entries: [String: ModelAllowEntry]) async throws {
    var config = try await fileAPI.readOpenClawConfig()
    config.agents.defaults.models = entries
    try await fileAPI.writeOpenClawConfig(config)
    try await fileAPI.reloadGateway()
}

// Example: limit to Opus and Sonnet with friendly aliases
try await setAllowlist([
    "anthropic/claude-opus-4-6":   ModelAllowEntry(alias: "Opus"),
    "anthropic/claude-sonnet-4-5": ModelAllowEntry(alias: "Sonnet"),
])
```

> **Note:** An empty or absent `agents.defaults.models` allows all models. Setting it to a non-empty map restricts in-session `/model` to only those entries.

### 5. Image model

If your app exposes image-processing workflows, you can let users set the image model separately:

```swift
func setImageModel(providerModel: String) async throws {
    var config = try await fileAPI.readOpenClawConfig()
    config.agents.defaults.imageModel.primary = providerModel
    try await fileAPI.writeOpenClawConfig(config)
    try await fileAPI.reloadGateway()
}
```

Typical image-capable models: `openai/gpt-4o`, `anthropic/claude-opus-4-6`, `openai/gpt-5.4`.

## Filtering the picker by connected providers

Show only models whose provider the user has already connected:

```swift
func availableModels(connectedProviders: Set<String>) -> [ModelEntry] {
    modelCatalog.filter { connectedProviders.contains($0.provider.lowercased()) }
}
```

Call `availableModels` with the provider IDs read from the user's `openclaw.json` env block (the same keys set by the quick-connect flows in `ios-managed-app.md`).

## Model ref format

Model refs follow the `provider/model-id` format. A few edge cases:

- **OpenRouter models** include a second `/`: use the full `openrouter/org/model-id` form (e.g., `openrouter/anthropic/claude-sonnet-4-5`).
- **Provider aliases** (e.g., `z.ai/*`) normalize to their canonical form (`zai/*`) internally; use the canonical form in config.
- Model refs are case-insensitive; OpenClaw normalizes to lowercase.

## Security notes

- Model switching is a configuration change, not a credential operation — no additional key vaulting is required.
- Restrict the allowlist in managed deployments to avoid users switching to expensive frontier models unexpectedly.
- The file-API bridge validates the Supabase JWT on every write; unauthenticated model-switch attempts are rejected at the file-api layer.
