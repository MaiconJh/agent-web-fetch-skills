# Fallback Strategies & Circuit Breaker Logic

## 1. Request Pipeline Flowchart

Start Analysis
      |
      v
Fetch git/trees?recursive=1 ---[404]---> Check 'master' / default_branch
      |
   [200 OK] (Store file path + sha)
      |
      v
Locate target path in tree[]
      |
 +----+----------------------------------------------+
 | Path Found                                        | Path Unknown (Need Symbol)
 |                                                   v
 |                                           Query Sourcegraph Stream
 |                                                   |
 |                                            [Empty / Failure]
 |                                                   v
 |                                           Query searchcode.com
 |                                                   |
 |                                            [Empty / Failure]
 |                                                   v
 |                                           GitHub Search API (Token Only)
 |                                                   |
 +-------------------+-------------------------------+
                     |
                     v
             Is it UI / Markup?
             (.tsx, .jsx, .vue)
               /            \
           [YES]            [NO]
             |                |
             v                v
   Fetch git/blobs/{sha}   Fetch raw.githubusercontent.com
   (Base64 - Safe JSX)     (Plain text)
