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
   - Ranks programs only on source-backed data (published payout ranges, disclosed response stats). Where a figure has no source, record it as unknown - never guess.

2. **Scope gate**
   - The only path from any worker to the network.
   - Validates a target URL/host against the stored in-scope patterns of one chosen program. Refuses everything else and logs the refusal. No override flag in v1.
   - Stores each program's explicit testing-policy text and its source URL. Automated workers run only when that policy explicitly permits automation; joining a program is never treated as permission. Unclear policy means manual-only.
   - Follows and validates every redirect hop: the final destination must also be in scope or the request is blocked.

3. **IDOR worker**
   - Input: program name, base URL, credentials/session for TWO of the user's own test accounts.
   - Maps object IDs from account A's own data (IDs in API responses and pages), replays the same read requests as account B, flags discrepancies where B receives A's data. It detects read access only; it never tests modification.
   - GET-only in v1: no writes, no deletes, no state changes of any kind, no exceptions.
   - Rate-limited, robots-aware, stops on WAF/block responses. Credentials, session tokens and cookies are redacted before any request or response is logged; third-party data is redacted from evidence. Output: candidate findings with sanitized request/response evidence.

4. **Extra workers** (pluggable, each behind the scope gate)
   - Exposed secrets/config in public JS of in-scope assets.
   - Subdomain takeover checks for in-scope domains (DNS-only, no exploitation).
   - Reflected-input checks with harmless canary strings.

5. **Reporter**
   - Turns a confirmed finding into a platform-ready draft: title, severity guess, steps to reproduce, impact, evidence bundle.
   - Estimates a payout band only from source-backed disclosed reports for that platform; unknown when no source exists. Estimates are never presented as promised or expected awards.
   - Queues the draft for human approval. The tool never submits anything.

## Hard rules (enforce in code, not just docs)

- Test only assets in scope of a program whose published policy the user is authorized under, only with accounts the user owns. Joining a program is not authorization by itself.
- Public scope is not authorization by itself. Many programs prohibit or limit automated scanning even for in-scope assets. The scope gate stores each program's automation rules and blocks automated workers unless the program's own policy allows automation; manual-only programs get a manual checklist, not a scan. When the policy is unclear, the gate treats it as manual-only and asks.
- No auto-submission, ever. A human files every report.
- No data exfiltration: credentials, session tokens and cookies are redacted before logging, and evidence proves the access bug with other people's data redacted.
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
