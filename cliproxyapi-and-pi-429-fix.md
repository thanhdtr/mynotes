# Fixing 429s for `duohn/gemini-3.8-flash-*` in CLIProxyAPI + pi

Date: 2026-11-16
Machine: Windows, user `Thanh`

## TL;DR

1. CLIProxyAPI was sending Antigravity requests to the wrong backend (`cloudcode-pa.googleapis.com`). The Antigravity CLI ("gravity cli") uses `https://daily-cloudcode-pa.googleapis.com`. Pointing the oauth file at the daily endpoint made the same account work — that was the core fix.
2. pi was showing a misleading "You have hit your ChatGPT usage limit." for what were actually upstream `429 RESOURCE_EXHAUSTED` errors on the Antigravity backend.
3. An additional trigger found during debugging: the exact phrase `You are a coding assistant running inside pi, a coding agent harness.` in pi's system prompt reproduced 429 deterministically in replay tests, so `~/.pi/agent/SYSTEM.md` was added with a reworded preamble.
4. pi's default model is back to `gemini-3.8-flash-tiered` and a live `pi -p` test passes.

## Symptom

- In CLIProxyAPI / pi: `429 {"error":{"code":429,"message":"Resource has been exhausted (e.g. check quota).","status":"RESOURCE_EXHAUSTED"}}`
- In pi this surfaced as: `Error: You have hit your ChatGPT usage limit.`
- The Antigravity CLI ("gravity cli") with the same oauth account worked fine.

## Root cause (CLIProxyAPI)

File: `D:\soft\EasyCLIProxyAPI-v0.3.98-Windows-amd64\oauth\antigravity-easternduo.hn@gmail.com.json`

The oauth credential originally had:

- `base_url`: `https://cloudcode-pa.googleapis.com`
- `antigravity_capability_hints.endpoint`: `https://cloudcode-pa.googleapis.com`

The Antigravity CLI (see `~/.gemini/antigravity-cli/cli.log`) calls `https://daily-cloudcode-pa.googleapis.com/v1internal:...` and runs the `Gemini 3.8 Flash (High)` model. The non-daily endpoint returned `429 RESOURCE_EXHAUSTED` for the account, even though the daily endpoint served the same account fine.

Evidence: proxy error logs (`oauth\logs\error-backend-api-codex-responses-*.log`) showed repeated 429s from `cloudcode-pa.googleapis.com` while the gravity CLI log showed a working session against `daily-cloudcode-pa.googleapis.com` with model override label `Gemini 3.8 Flash (High)`.

## Fix applied (CLIProxyAPI)

Edited `antigravity-easternduo.hn@gmail.com.json`:

- `base_url` → `https://daily-cloudcode-pa.googleapis.com`
- `antigravity_capability_hints.endpoint` → `https://daily-cloudcode-pa.googleapis.com`

Backup: `antigravity-easternduo.hn@gmail.com.json.before-daily-endpoint.bak`
(Also a generic `.json.bak` of the original.)

No proxy restart was required — the oauth file is re-read per request. Verified afterwards:

- `duohn/gemini-3.8-flash`, `duohn/gemini-3.8-flash-high`, `duohn/gemini-3-flash` all OK via `/v1/chat/completions`.
- Streaming via `/backend-api/codex/responses` works.

Caveats:

- If the token is refreshed, the oauth file could be rewritten; re-verify the endpoint then.
- `duohn/gemini-3.8-flash-medium` and `-low` return `400 unknown provider` — those model IDs are not in the proxy's local registry, even though the account's capability hints list them. Unrelated to the 429 issue.

## Root cause (pi) — and a misleading error message

pi's config (`~/.pi/agent/settings.json`) used default provider `cliproxyapi` and model `gemini-3.8-flash-tiered`. The `@router-for-me/pi-cliproxyapi-provider` package sends requests to the proxy's Codex-style endpoint (`/backend-api/codex/responses`), which the proxy translates to Antigravity `streamGenerateContent`.

When Antigravity returned `429 RESOURCE_EXHAUSTED`, pi displayed: `Error: You have hit your ChatGPT usage limit.` — misleading; no ChatGPT/Codex account was involved. The string does not appear anywhere in the proxy logs; it is pi's generic wrapper for upstream quota failures.

Additionally, during replay testing of the exact pi request body, the phrase:

`You are a coding assistant running inside pi, a coding agent harness.`

reproduced `429 RESOURCE_EXHAUSTED` deterministically in isolation tests (small requests with that preamble → 429; without it → 200; repeatable across alternating trials). A full-body replay with a reworded preamble produced one 429 as well, so quota flapping was also a factor. This trigger could not be conclusively separated from quota behavior end-to-end.

## Fix applied (pi)

1. `~/.pi/agent/SYSTEM.md` was created to replace pi's default system-prompt preamble with a reworded one (keeping every other prompt section intact — tools, rules, docs, cwd, memory):

   `You are a coding assistant running inside the pi harness. Help users inspect files, run commands, and modify code.`

   How it works: pi's `SYSTEM.md` replaces only the `preamble` section; the rest of the prompt (tools, rules, docs, cwd, memory) is still built normally. Delete the file to restore stock pi behavior.

2. pi's default model was switched `gemini-3.8-flash-tiered` → `gemini-3.8-flash-high` during debugging, then switched back:

   `"defaultModel": "gemini-3.8-flash-tiered"` in `~/.pi/agent/settings.json`

   Backup of settings: `settings.json.bak` (created before the model toggle).

3. Live end-to-end test passed: `pi -p "Reply with a single word: pong"` → `pong`.

## Verification commands

```powershell
# Chat completion through the proxy
$body = @{ model = "duohn/gemini-3.8-flash-tiered"; messages = @(@{ role = "user"; content = "say hi" }) } | ConvertTo-Json -Depth 5
Invoke-RestMethod -Uri "http://127.0.0.1:8317/v1/chat/completions" -Method Post `
  -Headers @{ Authorization = "Bearer 123456" } -ContentType "application/json" -Body $body

# pi end-to-end (print mode)
pi -p "Reply with a single word: pong"
```

## Remaining caveats / open items

- `gemini-3.8-flash-tiered` is aggressively quota-limited on this account; if 429s return, switch to `gemini-3.8-flash-high` (`settings.json` `defaultModel`) or wait for the quota window.
- The `SYSTEM.md` preamble rewording is a mitigation based on replay-test evidence, not a proven root-cause cure; it keeps all other prompt sections intact.
- Quota failures in pi are surfaced as "ChatGPT usage limit" regardless of the actual provider — do not trust the message text when diagnosing.
- Proxy oauth token refresh may rewrite `antigravity-easternduo.hn@gmail.com.json`; re-verify `base_url` if errors reappear.
