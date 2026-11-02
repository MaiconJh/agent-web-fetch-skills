# API Endpoints & Limit Specifications

## 1. GitHub Git Trees API
- Endpoint: GET https://api.github.com/repos/{owner}/{repo}/git/trees/{branch}?recursive=1
- Auth: None required (Personal Access Token optional via Authorization: Bearer <token>).
- Rate Limit: 60 requests/hour per IP (5,000/hour authenticated).
- Payload Bound: Up to 100,000 entries (or ~7 MB JSON max). Truncated trees contain "truncated": true.
- Usage: Primary structural discovery. Resolves the full project hierarchy and file SHAs in 1 request.

## 2. Raw GitHub User Content
- Endpoint: GET https://raw.githubusercontent.com/{owner}/{repo}/{branch}/{path}
- Auth: None.
- Rate Limit: Unmetered CDN delivery under standard web scraping patterns.
- Encoding: Plain text (no Base64 decoding required).
- Warning: Subject to HTML/JSX stripping by certain AI web fetch converters.
- Usage: Target file inspection for non-markup files (e.g., config, pure logic, css).

## 3. GitHub Git Blobs API (Markup Safe)
- Endpoint: GET https://api.github.com/repos/{owner}/{repo}/git/blobs/{sha}
- Auth: None required (shares the 60 req/h IP pool, 5,000/h auth).
- Rate Limit: 60 requests/hour per IP.
- Encoding: Base64 JSON payload.
- Usage: Mandatory alternative for .tsx, .jsx, and .vue files when web scrapers discard className and JSX elements.

## 4. Sourcegraph Streaming Search
- Endpoint: GET https://sourcegraph.com/.api/search/stream?q={query}+repo:{owner}/{repo}&v=V3
- Auth: None.
- Rate Limit: Soft throttling; prone to sporadic timeouts or empty result sets during high traffic.
- Payload Format: Server-Sent Events (SSE). Requires scanning event lines for "path":"...".
- Usage: Fast code search across public indexed repositories without authentication.

## 5. searchcode.com API
- Endpoint: GET https://searchcode.com/api/codesearch_I/?q={query}
- Auth: None.
- Rate Limit: 200 requests/month; 10 requests/minute. Max 50 results per query.
- Constraints: No regular expression queries; does not restrict by single repo easily.
- Usage: Secondary fallback when Sourcegraph hangs or yields no index.

## 6. GitHub Code Search API
- Endpoint: GET https://api.github.com/search/code?q={query}+repo:{owner}/{repo}
- Auth: Strictly Required (Header Authorization: Bearer {token}).
- Rate Limit: 10 requests/minute (unauthenticated returns 401); 30 requests/minute (authenticated).
- Constraints: Default branch only; files > 350 KiB excluded; max 100 results returned.
