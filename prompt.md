# bountygang - build spec (single source of truth)

You are building **bountygang**, a local CLI that makes bug bounty hunting as easy as possible: a gang of agents where one checks bounty sites and program scope, workers test each in-scope area (IDOR first), and a final agent produces a report draft ranked by expected payout. The tool assists an authorized hunter; it is not an auto-hacker.

## Stack

- Python 3.11+, `typer` CLI, `httpx`, `pydantic`, stdlib `sqlite3`, `pytest`.
- Playwright optional, only for JS-heavy in-scope pages.
- Runs on Windows PowerShell. Setup steps in README must be PowerShell-first.
- $0 default: local model via Ollama for parsing/summarizing. API keys optional in `.env`, off by default. `.env.example` committed, never a real `.env`.

## Agents

1. **Scout**
   - Pulls program lists and scope from HackerOne, Intigriti, Bugcrowd, Google Bughunters, huntr (public APIs/pages where allowed; otherwise a manual import command takes a saved JSON/markdown file).
   - Stores per program: name, platform, in-scope patterns, out-of-scope patterns, payout range, response stats, rules URL, last refreshed.
   - Ranks programs: payout band, fresh scope, response time, low competition.

2. **Scope gate**
   - The only path from any worker to the network.
   - Validates a target URL/host against the stored in-scope patterns of one chosen program. Refuses everything else and logs the refusal. No override flag in v1.

3. **IDOR worker**
   - Input: program name, base URL, credentials/session for TWO of the user's own test accounts.
   - Maps object IDs from account A's own data (IDs in API responses and pages), replays the same requests as account B, flags discrepancies where B can read or modify A's objects.
   - GET-only in v1. No deletes, no writes except against objects the tool itself created in the test account. Redact any third-party data in evidence.
   - Rate-limited, robots-aware, stops on WAF/block responses. Every request and response logged.

4. **Extra workers** (pluggable, each behind the scope gate)
   - Exposed secrets/config in public JS of in-scope assets.
   - Subdomain takeover checks for in-scope domains (DNS-only, no exploitation).
   - Reflected-input checks with harmless canary strings.

5. **Reporter**
   - Turns a confirmed finding into a platform-ready draft: title, severity guess, steps to reproduce, impact, evidence bundle.
   - Benchmarks expected payout against disclosed reports for that platform.
   - Queues the draft for human approval. The tool never submits anything.

## Hard rules (enforce in code, not just docs)

- Test only assets in scope of a program the user joined, only with accounts the user owns.
- No auto-submission, ever. A human files every report.
- No data exfiltration: evidence proves the access bug and redacts other people's data.
- Respect rate limits and program rules; stop on any block.
- $0 spend by default; ask before any paid API call.
- All runs logged locally (sqlite + JSONL).

## CLI

```
bountygang scout refresh            # pull/update programs
bountygang scout list               # ranked programs
bountygang scope add <program>      # import/confirm scope
bountygang check idor --program X --target URL
bountygang findings                 # candidate findings
bountygang report <finding-id>      # generate report draft
```

## Build order

1. Repo skeleton + scope gate + program store (sqlite).
2. Scout for one platform (HackerOne) + manual scope import.
3. IDOR worker.
4. Reporter.
5. Extra workers.

## Tests

pytest with mocked HTTP. No live targets in tests. Cover: scope gate allow/refuse, ID diffing logic, report draft generation, rate limiter, redaction.
