# Runbook: Fixing 429 (`RESOURCE_EXHAUSTED`) for Antigravity models via CLIProxyAPI and pi

Portable guide for an agent working on any machine. Diagnosis-first: do not change config blindly — confirm the evidence matches before applying edits.

## Goal

Make `duohn/gemini-3.8-flash-*` (or any Antigravity-backed model with a prefix like `duohn/`) work through CLIProxyAPI, including from pi, which uses the proxy's Codex-style endpoint.

## Symptoms

- `429 {"error":{"code":429,"message":"Resource has been exhausted (e.g. check quota).","status":"RESOURCE_EXHAUSTED"}}` from `http://127.0.0.1:8317` (or wherever CLIProxyAPI listens).
- In pi: `Error: You have hit your ChatGPT usage limit.` — **misleading; no ChatGPT/Codex account is involved.** It is pi's generic display of upstream quota failures.
- The Antigravity CLI ("gravity cli", binary `agent`/`agy`, logs in `~/.gemini/antigravity-cli/cli.log`) works fine with the same oauth account.

## Architecture recap

- CLIProxyAPI (aka EasyCLIProxyAPI, e.g. install dir `D:\soft\EasyCLIProxyAPI-<ver>-Windows-amd64\`, core `cpa-core/cli-proxy-api.exe`) exposes OpenAI-compatible endpoints (`/v1/chat/completions`) and a Codex-style endpoint (`/backend-api/codex/responses`).
- pi + `@router-for-me/pi-cliproxyapi-provider` sends Codex-style requests to the proxy; the proxy rewrites them to Antigravity (`cloudcode-pa` family, `v1internal:generateContent` / `:streamGenerateContent`).
- Antigravity oauth credentials live in the proxy's `oauth` directory (e.g. `<install>/oauth/antigravity-<email>.json`). The `prefix` field on that file is what enables `duohn/model` routing.
- pi is configured in `~/.pi/agent/settings.json` (`defaultProvider`, `defaultModel`). Model names without a prefix are resolved by the proxy (unprefixed may match multiple providers and round-robin).

## Step 1 — Confirm it is the known issue (not generic quota exhaustion)

1. Find the proxy's oauth dir and read the Antigravity credential file:
   - PowerShell: `Get-ChildItem "<install>\oauth"`
   - Check `base_url` and `antigravity_capability_hints.endpoint`.
   - If they are `https://cloudcode-pa.googleapis.com` while the Antigravity CLI log uses `daily-cloudcode-pa.googleapis.com`, this is the known backend-mismatch bug. **This was the actual root cause on the author's machine.**
2. Check the gravity CLI log to see which endpoint and model label it uses:
   - `~/.gemini/antigravity-cli/cli.log` → look for `URL: https://...` and lines like `Propagating selected model override to backend: label="Gemini 3.8 Flash (High)"`.
3. Read proxy error logs (`<oauth>/logs/error-*.log`): they show the real upstream URL, auth, and status. `=== API REQUEST ===` / `=== API RESPONSE ===` sections are ground truth. A working path shows `200` with an SSE stream; the bug shows `429` from `cloudcode-pa.googleapis.com`.

## Step 2 — Fix the backend endpoint (CLIProxyAPI)

Edit `<oauth-dir>/antigravity-<email>.json` (back up first):

- `base_url` → `https://daily-cloudcode-pa.googleapis.com`
- `antigravity_capability_hints.endpoint` → `https://daily-cloudcode-pa.googleapis.com`

No restart needed; the oauth file is re-read per request. Re-verify with a small request:

```powershell
$body = @{ model = "duohn/gemini-3.8-flash-high"; messages = @(@{ role = "user"; content = "say hi" }) } | ConvertTo-Json -Depth 5
Invoke-RestMethod -Uri "http://127.0.0.1:8317/v1/chat/completions" -Method Post `
  -Headers @{ Authorization = "Bearer <proxy-api-key>" } -ContentType "application/json" -Body $body
```

Notes:

- Some model IDs (`gemini-3.8-flash-medium`, `-low`) may return `400 unknown provider` — not a 429; the proxy's local registry doesn't know them. Use `-high` or `-tiered`, or add/upgrade the registry.
- Token refresh could rewrite this file; re-check after re-auth.

## Step 3 — Diagnose pi specifically

1. Read `~/.pi/agent/settings.json`: `defaultProvider` should be `cliproxyapi`; `defaultModel` the unprefixed Antigravity model id (the provider adds the `duohn/` prefix where applicable).
2. Run a non-interactive test: `pi -p "Reply with a single word: pong"`.
3. If it fails with `You have hit your ChatGPT usage limit.`, open the newest `error-backend-api-codex-responses-*.log` in the proxy oauth logs — it contains the exact request body pi sent (`"model":"..."`) and the upstream status. Replay that body with the tools below to bisect what triggers the 429:
   - Small `say hi` body via `/backend-api/codex/responses` works? Transport fine.
   - Add pi's `instructions` (system prompt). The author observed:
     - Isolated tests: the exact phrase `You are a coding assistant running inside pi, a coding agent harness.` reproduced 429 deterministically; reworded preamble -> 200.
     - Full-body replay with reworded preamble yielded one 429 → quota flapping is also possible; treat trigger-phrase fix as mitigation, not proven cure.
   - pi sends zstd-compressed bodies (`Content-Encoding: zstd`); Node ≥ 22 can reproduce with `zlib.zstdCompressSync`.

## Step 4 — Mitigations for pi

1. Replace the system-prompt preamble via `~/.pi/agent/SYSTEM.md`. pi treats `<agent-dir>/SYSTEM.md` as the replacement for its default **preamble** only; all other sections (tools, rules, docs, cwd, memory) are still built normally. Minimal reworded preamble that avoids the trigger phrase:

   `You are a coding assistant running inside the pi harness. Help users inspect files, run commands, and modify code.`

   (Delete the file to restore stock pi behavior.)
2. If `gemini-3.8-flash-tiered` keeps 429ing (it is quota-limited aggressively), set `"defaultModel": "gemini-3.8-flash-high"` in `~/.pi/agent/settings.json`. The Antigravity CLI's working session used `Gemini 3.8 Flash (High)`.
3. Lower the proxy's retry storm if quota is near the edge: `request-retry` in `cpa-core/config.yaml` (default 3; each pi attempt = 4 upstream calls, which can trip per-minute quota). Changing it requires restarting EasyCLIProxyAPI.
4. Restart pi after config changes; test with `pi -p`.

## Verification

```powershell
# Direct via proxy
curl.exe -s -N -X POST "http://127.0.0.1:8317/backend-api/codex/responses" `
  -H "Authorization: Bearer <proxy-api-key>" -H "Content-Type: application/json" `
  -H "Openai-Beta: responses=experimental" --data-binary "@payload.json"

# End to end via pi
pi -p "Reply with a single word: pong"
```

Expected: HTTP 200 / streaming SSE `response.completed`; pi prints the reply.

## Honesty notes / pitfalls

- The "ChatGPT usage limit" message is a red herring — always read the proxy's `=== API RESPONSE ===` log before trusting pi's error text.
- Single small requests can pass while real pi requests 429 (retry storms trip per-minute quotas; also a content-based trigger was observed). Test with the actual captured failing body, not with a toy request.
- `SYSTEM.md` is belt-and-braces; the endpoint fix (Step 2) is the primary repair. The exact trigger phrase in the preamble was observed to toggle 429/200 deterministically, but this could not be conclusively separated from quota windows end-to-end.
- Do not confuse `401/403` (auth) with `429` (quota/backend) — different fixes.
