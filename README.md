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
- [Single-Repo Stable Release Flow](#single-repo-stable-release-flow)
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

1. Creates a source tag and source release in `ai-hermes/wechat-mem0` (manual stable release flow).
2. Checks out the private source repository by tag.
3. Bumps the version to match the release tag.
4. Installs dependencies and resolves platform-native packages.
5. Builds platform-specific Electron installers.
6. Signs and notarizes macOS binaries using Apple Developer credentials.
7. Uploads all artifacts to a GitHub Release in this build repository.

---

## Releases

| Channel | Trigger | Tag |
|---------|---------|-----|
| **Stable (recommended)** | `workflow_dispatch` with `version` + `source_branch` | `v1.2.3` |
| **Stable (legacy)** | Push of a `v*.*.*` tag (uses `main` as source branch) | `v1.2.3` |
| **Nightly** | Scheduled daily at 04:00 CST (UTC 20:00) | `nightly` |

Download the latest installer from the
[Releases](../../releases/latest) page.

---

## Single-Repo Stable Release Flow

Use only this repository (`ai-hermes/wechat-mem0-build`) to publish a stable release:

1. Open **Actions → Build Private Repository → Run workflow**.
2. Set:
   - `version`: SemVer tag like `v1.2.3` or `v1.2.3-alpha.25`
   - `source_branch`: source branch in `ai-hermes/wechat-mem0` (default `main`)
3. Run workflow.

What the workflow does:

1. Resolves the latest commit on `source_branch` in `ai-hermes/wechat-mem0`.
2. Fails fast if `version` already exists in `ai-hermes/wechat-mem0`.
3. Creates the same tag and auto-generated release in `ai-hermes/wechat-mem0`.
4. Builds installers from that exact source tag.
5. Publishes release `version` in `ai-hermes/wechat-mem0-build` with auto-generated notes and artifacts.

For **legacy tag-push** releases, the workflow uses `main` as `source_branch`.
If the same tag already exists in `ai-hermes/wechat-mem0`, source tag/release
creation is skipped and the existing source tag is used for the build.

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
manual stable dispatch / tag push / schedule
        │
        ├── manual stable dispatch:
        │     create tag + release in ai-hermes/wechat-mem0
        │
        │
        ▼
Checkout ai-hermes/wechat-mem0 by resolved ref/tag (via PRIVATE_REPO_PAT)
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
