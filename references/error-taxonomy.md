# Error Taxonomy and Diagnostic Recovery

| HTTP Status or Condition | Root Cause | Immediate Agent Recovery Step |
|---|---|---|
| Stripped JSX or Missing Tailwind | Web fetch parser converted .tsx/.jsx into Markdown, discarding tags and className attributes. | Do not guess styling. Query GET https://api.github.com/repos/{owner}/{repo}/git/blobs/{sha} and decode Base64 content to recover exact JSX. |
| 401 Unauthorized (GitHub Search) | Attempted to use /search/code without a valid personal access token. | Do not retry. Immediately trip circuit breaker and report authentication requirement. |
| 403 Forbidden or 429 Too Many Requests | Exceeded IP rate limit on GitHub API (60/h unauthenticated). | Stop tree/blob requests. Fall back to direct guesses via raw.githubusercontent.com for standard files (README.md, package.json). |
| 404 Not Found (git/trees) | Incorrect branch name (main vs master) or private repository. | Execute Branch Resolution Pattern. If default branch fails, notify user that repository may be private. |
| 404 Not Found (raw) | Path does not exist or target is a directory. | Verify path against initial tree manifest. Never issue raw fetches for directory objects. |
| Socket Timeout or Empty SSE (Sourcegraph) | Upstream indexing cold start or network flake. | Run 1 backoff retry. If unresponsive within 15 seconds, proceed directly to searchcode. |
| Quota Depleted (searchcode) | 200 requests/month ceiling reached. | Stop querying searchcode for active session; output audit log requesting token or manual path pointer. |
