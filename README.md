# RethinkAI — Build Repository

This repository contains the public CI/CD pipeline that builds and releases
[RethinkAI](https://github.com/ai-hermes/wechat-mem0) — a cross-platform
Electron desktop application that brings persistent AI memory to your daily
workflow.

> **Note:** Application source code lives in the private repository
> [`ai-hermes/wechat-mem0`](https://github.com/ai-hermes/wechat-mem0).
> This repo exists solely to produce signed, notarized installers and publish
> them as GitHub Releases.

---

## Table of Contents

- [Overview](#overview)
- [Releases](#releases)
- [Build Matrix](#build-matrix)
- [CI/CD Workflow](#cicd-workflow)
- [Required Secrets](#required-secrets)
- [Code Signing Policy](#code-signing-policy)
- [Privacy Policy](#privacy-policy)
- [License](#license)

---

## Overview

RethinkAI is an AI-powered desktop assistant that integrates long-term memory
(via [Mem0](https://mem0.ai)) with large language models, web search
([Tavily](https://tavily.com)), and analytics
([PostHog](https://posthog.com)).  The build pipeline in this repository:

1. Checks out the private source repository.
2. Bumps the version to match the release tag (when triggered by a tag push).
3. Installs dependencies and resolves platform-native packages.
4. Builds platform-specific Electron installers.
5. Signs and notarizes macOS binaries using Apple Developer credentials.
6. Uploads all artifacts to a GitHub Release.

---

## Releases

| Channel | Trigger | Tag |
|---------|---------|-----|
| **Stable** | Push of a `v*.*.*` tag | `v1.2.3` |
| **Nightly** | Scheduled daily at 04:00 CST (UTC 20:00) | `nightly` |
| **Manual** | `workflow_dispatch` with a branch name | — |

Download the latest installer from the
[Releases](../../releases/latest) page.

---

## Build Matrix

| Platform | Runner | Output |
|----------|--------|--------|
| macOS | `macos-latest` | `.dmg`, `.zip` |
| Windows | `windows-latest` | `.exe`, `.blockmap` |

Linux builds are prepared but currently disabled in the matrix.

---

## CI/CD Workflow

The single workflow file
[`.github/workflows/build_private_repo.yml`](.github/workflows/build_private_repo.yml)
drives everything.

```
tag push / schedule / manual dispatch
        │
        ▼
Checkout ai-hermes/wechat-mem0  (via PRIVATE_REPO_PAT)
        │
        ▼
Setup Node 22 + npm ci
        │
        ├── macOS ──► npm run build:mac  ──► sign & notarize ──► .dmg / .zip
        │
        └── Windows ► npm run build:win ──────────────────────► .exe
        │
        ▼
softprops/action-gh-release  →  GitHub Release
```

On a **nightly** run, existing release assets are cleaned up before uploading
fresh ones so the `nightly` tag always points to today's build.

---

## Required Secrets

Configure the following secrets in **Settings → Secrets and variables →
Actions** before running the workflow.

| Secret | Purpose |
|--------|---------|
| `PRIVATE_REPO_PAT` | Personal Access Token to clone the private source repo and publish releases |
| `LLM_BASE_URL` | Base URL of the LLM provider API |
| `LLM_API_KEY` | API key for the LLM provider |
| `LLM_MODEL_NAME` | Model identifier (e.g. `claude-sonnet-4-6`) |
| `TAVILY_API_KEY` | Tavily web-search API key |
| `POSTHOG_HOST` | PostHog ingestion host |
| `POSTHOG_API_KEY` | PostHog project API key |
| `DMX_API_TOKEN` | DMX service token |
| `DMX_API_USER` | DMX service user |
| `MAC_CERTS` | Base-64-encoded `.p12` macOS signing certificate |
| `MAC_CERTS_PASSWORD` | Password for `MAC_CERTS` |
| `APPLE_ID` | Apple ID used for notarization |
| `APPLE_APP_SPECIFIC_PASSWORD` | App-specific password for notarization |
| `APPLE_TEAM_ID` | Apple Developer Team ID |

---

## Code Signing Policy

Free code signing provided by [SignPath.io](https://signpath.io),
certificate by [SignPath Foundation](https://signpath.org).

### Team roles

- **Committers and reviewers:**
  [Members](https://github.com/orgs/ai-hermes/teams/members)
- **Approvers:**
  [Owners](https://github.com/orgs/ai-hermes/people?query=role%3Aowner)

### Signing scope

| Platform | Method |
|----------|--------|
| macOS | Apple Developer ID certificate — signed & notarized via `electron-builder` |
| Windows | SignPath.io Authenticode signing |

Binaries that have not been signed by one of the certificates listed above
should be treated as unofficial and potentially unsafe.

---

## Privacy Policy

This program will not transfer any information to other networked systems
unless specifically requested by the user or the person installing or
operating it.

Data collected by optional integrations (PostHog analytics) is governed by
their respective privacy policies.  Analytics can be disabled at any time
from within the application settings.

---

## License

See the [LICENSE](LICENSE) file for details.
Build tooling and workflow scripts in this repository are released under the
[MIT License](https://opensource.org/licenses/MIT).
