# TODO (state as of 2026-09-13 13:10, pushed unreviewed at the user's request)

Branch `node-cli` = master now. Node port done by a Codex `sol` worker, docs by an Antigravity Opus worker;
Linux smoke passed (agy run/resume/close, claude haiku run). **No review pass completed** — every vendor
ran out of quota mid-review. Open items, in priority order:

1. **Review the Node port** against `docs`-less brief items: safety (Codex never gets
   `--dangerously-bypass-approvals-and-sandbox`; `install --statusline` must chain, never clobber, an
   existing statusline command), queue handshake races, interrupt kills the whole process tree.
2. **Bug: agy quota/stderr errors return exit 0.** Antigravity prints `error: Individual quota reached…`
   on stderr, not in the stream-json; `lib/runner.js` only flags `type: error` events. Treat a non-empty
   stderr `error:` line + empty result as failure (exit 1), like Grok 402.
3. **Codex smoke of the Node CLI not run** (Codex 5h window was exhausted). Run:
   `DELEGATE_OUT=/tmp/x node bin/cli.js run codex luna brief.md --cd /tmp/x --effort low`, then `resume`, then `close`.
4. **macOS / Windows paths untested** (Terminal.app via osascript; `wt.exe` / `cmd /k`; junction→copy fallback).
5. Renderer: confirm claude stream-json tool_use/tool_result rendering on a real run with tools.
6. Quota lesson to keep in docs: Antigravity's Claude/GPT weekly bucket is ~2 Opus-sized jobs; Gemini flash
   uses ~250k input tokens per job.

## Notes (fork: fiazulhaquealt-netizen — 2026-09-14)

Design / product notes only (no code yet). Captured from review + multi-vendor quota discussion.

### A. Cross-vendor session continuity (resume across vendors)

Today `resume` is **same-vendor only** (Claude conversation ID / Codex thread / agy session). When a
provider dies on quota, skills say “reroute” = spawn a **new** worker — prior thread context is lost
unless the orchestrator re-pastes.

**Preferred model:** do **not** try to port proprietary session IDs across CLIs (they aren’t portable).
Each job already has a durable `outDir` (`brief.md`, `prompt.md`, `events.jsonl`, `last.md`). On
quota death / forced vendor switch:

1. Keep the same job `outDir` as the source of truth.
2. Seed the next vendor’s prompt from that transcript (brief + last progress + open asks), not from
   a foreign session ID.
3. Treat this as **multi-vendor resume**: continue the *job*, not the vendor thread.

Same mechanism covers “resume in multi vendor” and “continuity on quota death.”

### B. Open OmniRoute / OpenRouter free-quota vendor

Candidate extra free lane for workers when Claude/Codex/agy/Grok windows are dry:

- OpenRouter free models + OmniRoute-style routing (~50 req/day free tier, or higher after small
  lifetime spend; rate limits e.g. ~20 rpm — verify current numbers before coding).
- Needs a new vendor adapter + `status` snapshot.
- Free request counts are **not** reliably exposed by API → local free-window counter (or equivalent)
  so the orchestrator can steer away before hard 429s.

Priority: lower than (A); only if free tier still exists and fits the terminal-worker model.

### C. Resume multi-vendor (explicit)

Same as (A): first-class flow so an orchestrator can `resume` a job onto another vendor when the
current one is exhausted, using `outDir` seeding. Document in `/delegates` skill once implemented.
