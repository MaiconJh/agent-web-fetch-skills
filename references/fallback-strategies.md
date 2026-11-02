# Fallback Strategies and Circuit Breaker Logic

## 1. Request Pipeline Flow

- Step 1: Start Analysis.
- Step 2: Fetch git/trees?recursive=1.
  - If HTTP 404: Check branch "master". If still 404, query repository default_branch.
  - If HTTP 200: Store tree manifest (both file path and sha).
- Step 3: Locate target path in tree manifest.
  - Case A (Path Found): Proceed to Step 4.
  - Case B (Path Unknown / Need Symbol):
    - Try Sourcegraph Stream.
    - If empty or timeout: Try searchcode.com.
    - If empty or failure: Try GitHub Search API (Token only).
    - If all fail: Emit Explicit Degradation Notice.
- Step 4: Determine Content Fetch Strategy:
  - If UI or Markup file (.tsx, .jsx, .vue): Fetch git/blobs/{sha} and decode Base64 to preserve JSX and Tailwind classes intact.
  - If Standard file (.json, .ts, .js, .md, .css): Fetch raw.githubusercontent.com directly as plain text.

## 2. Exponential Backoff with Jitter

For intermittent endpoints (Sourcegraph, searchcode):
- Attempt 1: Immediate call (timeout: 15s).
- Attempt 2: Wait 2s + jitter (between 100ms and 500ms).
- Attempt 3: Fail immediately to the next tier in the hierarchy.

## 3. Circuit Breaker States

- Sourcegraph:
  - Failure threshold: 3 non-200 or timed-out calls.
  - Action: Trip state to OPEN. Route directly to searchcode.
  - Reset timeout: 30 seconds.

- searchcode:
  - Failure threshold: 2 failures or HTTP 429.
  - Action: Trip state to OPEN. Do not exhaust monthly quota.
  - Reset timeout: 60 seconds.

- GitHub Search:
  - Failure threshold: 1 HTTP 401 or 2 HTTP 429s.
  - Action: Trip state to OPEN. Permanently skip for current session if 401.
  - Reset timeout: 120 seconds.

## 4. Branch Resolution Pattern

1. Issue initial query assuming branch=main.
2. If HTTP 404, query branch=master.
3. If HTTP 404, execute single metadata check: GET https://api.github.com/repos/{owner}/{repo} and extract "default_branch".
4. Re-issue git/trees query using the extracted default_branch.
