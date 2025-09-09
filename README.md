#### AFFiNE + LiteLLM “Responses → Chat” Adapter

## A tiny, production-friendly fix for connecting AFFiNE Copilot to LiteLLM/Ollama when AFFiNE streams to /v1/responses but your stack only supports /v1/chat/completions.


# Problem: AFFiNE (via the Vercel AI SDK) streams to /v1/responses and expects Responses-API SSE events. LiteLLM/Ollama speak OpenAI Chat Completions (/v1/chat/completions) and a different stream shape. Result:
" Provider openai failed with unexpected_response: text part … not found "

# Tried: Upgrading LiteLLM, reverse-proxy tricks, patching AFFiNE bundle to use Chat Completions. Too brittle/invasive.

# Solution: Drop in a small Node adapter that:
 - Accepts POST /v1/responses
 - Calls LiteLLM POST /v1/chat/completions
 - Normalizes AFFiNE inputs to valid OpenAI chat messages
 - Streams back Responses-API SSE events with a stable item_id
 - Passes everything else (models/embeddings) straight to LiteLLM

# Routing: Caddy sends only /v1/responses* to the adapter; everything else to LiteLLM.

# Note: In this example I am using a domain + ssl for both Affine and LiteLLM (running docker containers + ollama installed locally along)

### Why this is needed

# AFFiNE’s Copilot (Vercel AI SDK) uses an OpenAI-compatible Responses API. Its streaming contract requires a prelude:
 1. response.created
 2. response.output_item.added (declares the message item & id)
 3. response.content_part.added (declares the “text part” being streamed)
 4. response.output_text.delta … (many times)
 5. response.output_text.done
 6. response.content_part.done
 7. response.output_item.done
 8. response.completed

LiteLLM’s Chat Completions stream emits OpenAI chunks (choices[].delta.content). Without the Responses prelude, AFFiNE throws:
“text part … not found”.

The adapter bridges these two worlds.

### Architecture

AFFiNE UI
   │  baseURL: https://your-domain.com/v1
   ▼
Caddy (reverse proxy)
   ├─ /v1/responses*  ─────────►  responses-adapter:4011  ──►  LiteLLM :4000  ─►  Ollama
   └─ (everything else) ───────►  LiteLLM :4000

- Only /v1/responses is adapted.
- /v1/models, /v1/embeddings, /v1/chat/completions continue to hit LiteLLM directly.

responses-adapter/
├─ Dockerfile
├─ package.json
└─ server.js     ← the adapter (drop-in)
docker-compose.yml
Caddyfile

## Build
docker compose up -d --build responses_adapter
docker compose restart caddy

## Troubleshooting

ENOTFOUND litellm in adapter
The adapter isn’t on the same Docker network as LiteLLM. Attach both to affine_net (or use LITELLM_URL=http://caddy:80 to call through Caddy).

invalid content type in LiteLLM
Means AFFiNE sent content parts like {type:"input_text", text:"..."}. The adapter above normalizes to string content; ensure you’re running the updated server.js.

Still seeing “Claude” label in UI
It’s cosmetic. Disable Anthropic provider and clear browser site data; scenarios already target your  models. If that didn't solve it change it from affine's code directly.

CORS/stream stalls
Keep X-Accel-Buffering: no and avoid gzipping the SSE stream. The Caddy block above is safe.

## Why this approach

No invasive patches to AFFiNE’s bundle

Works with current LiteLLM and Ollama out of the box

Touches only one endpoint (/v1/responses) and keeps the rest of your routing intact

Clear, auditable behavior


# License / Notes

This adapter is intentionally minimal and can live inside your infra repo. Modify as needed (timeouts, logging, auth). It doesn’t persist data or log prompts by default.

# Result: AFFiNE Copilot streams cleanly via your OpenAI-compatible LiteLLM/Ollama stack, with stable SSE and zero model/provider lock-in.