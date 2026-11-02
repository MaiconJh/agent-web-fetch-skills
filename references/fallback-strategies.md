# Fallback Strategies & Circuit Breaker Logic

## 1. Request Pipeline Flowchart

Start Analysis
      |
      v
Fetch git/trees?recursive=1 ---[404]---> Check 'master' / default_branch
      |
   [200 OK]
      |
      v
Locate target path in tree[]
      |
 +----+---------------------------+
 | Path Found                     | Path Unknown (Need Symbol)
 v                                v
Fetch raw.githubusercontent.com   Query Sourcegraph Stream
 |                                |
 +--------------+-----------------+
                |
         [Empty / Failure]
                v
        Query searchcode.com
                |
         [Empty / Failure]
                v
        GitHub Search API (ONLY if GITHUB_TOKEN exists)
                |
         [All Exhausted]
                v
Emit Explicit Degradation Notice

## 2. Exponential Backoff with Jitter

For intermittent endpoints (Sourcegraph, searchcode):
- Attempt 1: Immediate call (timeout: 15s).
- Attempt 2: Wait 2s + rand(100ms, 500ms).
- Attempt 3: Fail immediately to the next tier in the hierarchy.

## 3. Circuit Breaker States

| Service | Failure Threshold | Action | Reset Timeout |
|---|---|---|---|
| Sourcegraph | 3 non-200 or timed-out calls | Trip state to OPEN. Route directly to searchcode. | 30 seconds |
| searchcode | 2 failures or HTTP 429 | Trip state to OPEN. Do not exhaust monthly quota. | 60 seconds |
| GitHub Search | 1 HTTP 401 or 2 HTTP 429s | Trip state to OPEN. Permanently skip for current session if 401. | 120 seconds |

## 4. Branch Resolution Pattern
1. Issue initial query assuming branch=main.
2. If HTTP 404, query branch=master.
3. If HTTP 404, execute single metadata check: GET https://api.github.com/repos/{owner}/{repo} and extract "default_branch".
4. Re-issue git/trees query using the extracted default_branch.
