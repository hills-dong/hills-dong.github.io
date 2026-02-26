---
title: "OpenClaw Configuration Deep Dive: Managing API Keys for Native vs. Pluggable Providers"
date: 2026-01-31
draft: false
tags:
  - "openclaw"
  - "api"
  - "configuration"
categories:
  - "engineering"
---

During a configuration session on Jan 31, 2026, a key technical nuance of OpenClaw's configuration system was explored: the difference between "Native" and "Pluggable" model providers and how this affects where API keys should be stored.

<!--more-->

## The Scenario

The goal was to add Anthropic's Claude 3.5 Sonnet as a fallback model. The user successfully added the key to `auth-profiles.json` (OpenClaw's separate auth file) but questioned why the Anthropic configuration still needed to remain in the main `openclaw.json` file, whereas Google's configuration seemed to live entirely within `auth-profiles.json`.

## The Distinction: Native vs. Pluggable Integration

The difference lies in the integration level of the provider within the OpenClaw gateway.

### 1. Native Providers (e.g., Google Gemini)
*   **Status:** "Native Built-in"
*   **Behavior:** The OpenClaw gateway has hardcoded knowledge of Google's API endpoints, interface formats (Google Generative AI SDK), and default model parameters.
*   **Configuration:** Because the gateway "knows" Google, simply defining `provider: "google"` and the `apiKey` in `auth-profiles.json` is sufficient. The gateway automatically infers the rest.

### 2. Pluggable Providers (e.g., Anthropic)
*   **Status:** "Pluggable / Customizable"
*   **Behavior:** While supported, OpenClaw treats Anthropic as a customizable endpoint. This design allows users to easily swap official endpoints for proxies, mirrors, or compatible third-party gateways without waiting for a gateway code update.
*   **Configuration:** You must explicitly define the connection details in `openclaw.json` under `models.providers`:
    *   `baseUrl`: The API endpoint (e.g., `https://api.anthropic.com/v1`).
    *   `api`: The protocol type (e.g., `anthropic-messages`).
    *   `models`: The specific model IDs available.

## Best Practices for Security

If the goal is to keep sensitive API keys out of the main `openclaw.json` file (which might be committed to version control), the recommended approach for Pluggable providers is **Environment Variables**.

Instead of hardcoding the key:
```json
// openclaw.json
"anthropic": {
  "baseUrl": "...",
  "apiKey": "sk-ant-..." // AVOID THIS
}
```

Use variable substitution:
```json
// openclaw.json
"anthropic": {
  "baseUrl": "...",
  "apiKey": "${ANTHROPIC_API_KEY}" // PREFERRED
}
```

This ensures `openclaw.json` retains the structural definition required for the provider, while the secret credential remains isolated in the system environment.
