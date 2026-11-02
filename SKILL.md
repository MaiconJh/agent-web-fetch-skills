---
name: github-repo-analysis
description: Analyzes any public GitHub repository via targeted web fetches. First anchors rules from the remote skills repository, then inspects the target repository using git/trees, raw content, and fallback code search. NEVER downloads whole repositories.
version: 5.1.0
tags: [github, api, primary, repository-analysis, code-search, web-fetch]
---

# GitHub Repository Analysis (Remote Hub Bootstrapper)

Analyzes, inspects, reviews, and debugs ANY public GitHub repository provided by the user using agent-native web fetching. MCP Local Files is strictly an optional secondary fallback.

## 🚨 CRITICAL RULE: NEVER DOWNLOAD FULL REPOSITORIES 🚨

NEVER use BDS:AUTO:REQUEST_GITHUB_FETCH. Full repository downloads flood the context buffer and destroy reasoning quality. Always execute surgical, selective endpoints using BDS:AUTO:REQUEST_WEB_FETCH.

---

## Step 0: Remote Anchoring (MANDATORY FIRST FETCH)

Before inspecting the user's target repository, perform exactly ONE fetch to pull the authoritative operational directives and API limits from the remote skills hub:

Target Hub URL:
https://raw.githubusercontent.com/MaiconJh/agent-web-fetch-skills/main/references/api-limits.md

Guidelines for Step 0:
- If already fetched earlier in the current conversation, reuse the cached memory and skip Step 0.
- If the fetch fails or times out, immediately fall back to the built-in protocols defined below without halting execution.

---

## Step 1: Ingest Target Repository Tree (Single Request)

Identify {owner} and {repo} from the user's prompt or target URL, then fetch the full hierarchy:

https://api.github.com/repos/{owner}/{repo}/git/trees/{branch}?recursive=1

Branch resolution sequence:
1. Default to branch "main".
2. If HTTP 404, immediately try branch "master".
3. If still 404, fetch https://api.github.com/repos/{owner}/{repo} and extract "default_branch".
4. Parse the returned JSON tree[] (path, sha, type, size). Retain the "sha" of each file for potential blob fetching. Do not issue sequential list_directory calls.

---

## Step 2: Target Path Resolution

Filter the paths from the retrieved tree[] to match the user's intent:
- Build & dependencies: package.json, Cargo.toml, go.mod, pom.xml, pyproject.toml, Dockerfile.
- Configuration: **/config*, **/auth*, **/.env.example, **/settings*.
- Core logic & UI: **/routes*, **/controllers*, **/services*, **/api*, **/src*, **/components*, **/views*.

---

## Step 3: Fetch Target File Content (Raw vs. Blob Rule)

Select the fetch strategy based on file extension to avoid HTML/Markdown stripping:

### Case A: Standard Files (.json, .ts, .js, .md, .yml, .css)
Fetch directly via raw CDN delivery:
https://raw.githubusercontent.com/{owner}/{repo}/{branch}/{path}

- Returns direct plain text without API rate limits.
- Never issue raw fetches for directory paths.

### Case B: UI Markup Files (.tsx, .jsx, .vue, .html) — CRITICAL PARSER BYPASS
Web fetch scrapers often strip JSX/HTML tags (like <div className="...">) when fetching raw files, destroying visual structure and Tailwind utility classes. For any UI file where classes, attributes, or layout structure must be preserved:

1. Locate the file's "sha" from the tree[] array collected in Step 1.
2. Fetch the raw Git Blob JSON:
   https://api.github.com/repos/{owner}/{repo}/git/blobs/{sha}
3. Read the "content" property (encoded in Base64) and decode it. Because Base64 is stored as a raw JSON string, the fetch tool cannot strip HTML/JSX tags, guaranteeing 100% faithful markup, Tailwind classes, and layout hierarchy.

---

## Step 4: Symbol & Code Search (Fallback Only)

If the tree structure does not reveal which file contains a required symbol or function:

1. Sourcegraph Stream (Timeout: 15s; 1 Retry):
   https://sourcegraph.com/.api/search/stream?q={query}+repo:{owner}/{repo}&v=V3
   Scan Server-Sent Events (SSE) for the "path" key. If unresponsive, advance to option 2.

2. searchcode.com (Secondary Fallback):
   https://searchcode.com/api/codesearch_I/?q={query}
   Filter results to match the target repository paths.

3. GitHub Search API (Token Mandatory):
   https://api.github.com/search/code?q={query}+repo:{owner}/{repo}
   SKIP entirely if no GITHUB_TOKEN or Authorization header is available (avoids 401).

---

## Resilience & Gotchas

- Network in CODE_RUNNER: The Python execution sandbox has no internet access. Do not use urllib or requests inside CODE_RUNNER; use BDS:AUTO:REQUEST_WEB_FETCH.
- JSX/Tailwind Stripping: Never use raw.githubusercontent.com for UI components if you notice structural markup disappearing. Immediately fall back to the git/blobs/{sha} Base64 endpoint.
- Exponential Backoff: Wait 2s on attempt 1, 4s + jitter on attempt 2 before advancing search tiers.
- Circuit Breaker: Cut off Sourcegraph after 3 failures (30s cooldown) and searchcode after 2 failures (60s cooldown).
- Graceful Report: If target files cannot be accessed, list all attempted endpoints and explain whether the failure was caused by rate limits, missing tokens, or private repository visibility.
