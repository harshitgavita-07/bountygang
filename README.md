# bountygang

A local multi-agent bug bounty assistant. A gang of agents does the grunt work: one watches bounty programs and their scope, workers test only what is in scope (IDOR first), and the last one turns confirmed findings into report drafts ranked by expected payout. You approve every report before anything is filed.

Status: spec stage. The whole build spec is `prompt.md`.

## The gang

1. **Scout** - pulls programs and scope from HackerOne, Intigriti, Bugcrowd, Google Bughunters, huntr. Ranks them by payout, fresh scope, response time, competition.
2. **Scope gate** - every check goes through it. If the target is not in scope of a program you joined, it refuses and logs the refusal.
3. **IDOR worker** - tests object IDs across two of your own test accounts and flags where account B can read account A's data. GET-only, no writes, credentials redacted before anything is logged.
4. **Extra workers** - pluggable checks, each behind the scope gate: exposed secrets in public JS, subdomain takeover (DNS only), harmless reflected-input canaries.
5. **Reporter** - drafts the platform-ready report (steps, impact, evidence), estimates payout only from published disclosed reports, queues it for your yes. Never submits.

## Hard rules (code, not vibes)

- Only assets in scope of a program you joined, only accounts you own.
- In-scope is not automatic permission: programs that ban automated scanning get a manual checklist, not a scan.
- No auto-submission, ever.
- No data exfiltration. Evidence is proof of the access bug, redacted.
- Respect rate limits and program rules. Stop on any block.
- $0 spend by default. Local model (Ollama) for parsing; paid APIs off unless you say so.
- Every run logged locally.

## Build it

Open this repo in Claude Code or Codex and paste one line:

```
Read prompt.md and build it.
```

The spec is the single source of truth. Changes get folded into `prompt.md`, not scattered across chat.
