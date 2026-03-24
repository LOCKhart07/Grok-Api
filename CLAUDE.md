# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Grok-Api is a reverse-engineered Python wrapper for Grok AI that works without API keys or authentication. It provides both a direct Python interface (`core.Grok`) and a FastAPI REST server (`api_server.py`).

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run the API server (development)
python api_server.py

# Run the API server (production)
uvicorn api_server:app --host 0.0.0.0 --port 6969 --workers 50

# Run the manual test script
python manual.py
```

There are no tests, linter, or type-checker configured.

## Architecture

### Request Flow

`Grok.start_convo()` orchestrates the full flow:

1. **Session init** (`_load`) — loads `grok.com/c`, extracts Next.js server action IDs and the XSID script path from bundled JS chunks, plus baggage/sentry-trace from meta tags.
2. **Anonymous auth** (3x `c_request`) — performs a 3-step handshake:
   - Step 0: sends ECDSA public key via multipart, gets `anonUserId`
   - Step 1: receives a challenge, signs it with the private key (`Anon.sign_challenge`)
   - Step 2: gets a verification token and animation data for XSID generation
3. **XSID signature** (`Signature.generate_sign`) — computes `x-statsig-id` header from the verification token, SVG path data, and script-derived index values
4. **Conversation request** — POSTs to `/rest/app-chat/conversations/new` (or `.../responses` for follow-ups), parses the NDJSON streaming response

### Continuation

`extra_data` returned in every response contains all state needed to continue a conversation (cookies, action IDs, conversation ID, parent response ID, private key). Pass it back to `start_convo()` to send follow-up messages — this skips step 0 of the handshake.

### Key Modules

- **`core/grok.py`** — `Grok` class, main entry point. Manages session, cookies, and the multi-step auth + conversation flow.
- **`core/reverse/anon.py`** — ECDSA key generation and challenge signing using `coincurve`.
- **`core/reverse/xctid.py`** — `Signature` class that generates the `x-statsig-id` token (SVG path parsing, cubic bezier easing, style simulation, SHA-256 HMAC).
- **`core/reverse/parser.py`** — Extracts server action IDs, verification tokens, and SVG animation data from HTML/JS. Caches parsed script data in `core/mappings/`.
- **`core/headers.py`** — Predefined header sets (`LOAD`, `C_REQUEST`, `CONVERSATION`) with header ordering via `fix_order`.
- **`core/runtime.py`** — `Run.Error` decorator (catch-all with exit) and `Utils.between` (string extraction).

### Caching

`core/mappings/txid.json` and `core/mappings/grok.json` cache parsed script data so repeated runs don't re-fetch/re-parse Grok's JS bundles. These files are auto-created and updated by `Parser`.

## Available Models

| Model ID | Mode |
|---|---|
| `grok-3-auto` | auto (default) |
| `grok-3-fast` | fast |
| `grok-4` | expert |
| `grok-4-mini-thinking-tahoe` | grok-4-mini-thinking |

## Important Notes

- Uses `curl_cffi` with Chrome 136 TLS fingerprint impersonation — standard `requests` will not work.
- The wrapper is fragile: any Grok web interface update can break action ID parsing, the auth handshake, or response format.
- Anti-bot rejections trigger automatic retry with a fresh session (recursive `start_convo` call).
- A proxy is required for the API server endpoint but optional for direct Python usage.
