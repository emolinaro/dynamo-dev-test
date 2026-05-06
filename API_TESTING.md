# Dynamo v1.1.0 API Smoke Tests

Use these checks after port-forwarding a frontend service or a GAIE gateway.
They cover the API changes called out in the Dynamo `v1.1.0` release notes from
May 4, 2026: `/v1/responses`, Anthropic-compatible `/v1/messages`, and
`context_window` metadata on `/v1/models`.

## Prerequisites

Set the target endpoint first:

```bash
export BASE_URL=http://127.0.0.1:8000
export MODEL=Qwen/Qwen3-0.6B
```

If you are testing GAIE instead of a direct frontend port-forward, add the
appropriate host header to each `curl` command below, for example:

```bash
-H "Host: gp-gaie-model-a.example.com"
```

## 1. Models

Use this to confirm the frontend is up and to inspect the model metadata that
`v1.1.0` now exposes through `/v1/models`:

```bash
curl -fsS "$BASE_URL/v1/models" | head -c 2000
echo
```

Look for the expected model ID and, when the backend reports it, a
`context_window` field.

## 2. OpenAI Chat Completions

```bash
curl -fsS "$BASE_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"${MODEL}\",
    \"messages\": [{\"role\":\"user\",\"content\":\"Say hello in one short sentence.\"}],
    \"temperature\": 0.2,
    \"max_tokens\": 64
  }" | head -c 2000
echo
```

## 3. OpenAI Responses

```bash
curl -fsS "$BASE_URL/v1/responses" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"${MODEL}\",
    \"input\": \"Say hello in one short sentence.\",
    \"max_output_tokens\": 64
  }" | head -c 2000
echo
```

## 4. Anthropic Messages

```bash
curl -fsS "$BASE_URL/v1/messages" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d "{
    \"model\": \"${MODEL}\",
    \"max_tokens\": 64,
    \"messages\": [
      {
        \"role\": \"user\",
        \"content\": [
          {\"type\": \"text\", \"text\": \"Say hello in one short sentence.\"}
        ]
      }
    ]
  }" | head -c 2000
echo
```

## v1.1.0 Notes

- `/v1/messages` is the Anthropic-compatible surface highlighted in the
  `v1.1.0` release notes for Claude Code and related clients. The current
  frontend docs still mark it as experimental, so treat this as a smoke test
  rather than a stability guarantee.
- `cache_control` is still accepted for compatibility on `/v1/messages`, but
  `v1.1.0` removes frontend cache pinning behavior. Treat it as request
  metadata, not sticky cache state.
- For multimodal Claude-style requests, `v1.1.0` can translate OpenAI-style
  `image_url` content into Anthropic image content on vLLM backends.
- The public docs may lag behind the release notes on some of these details, so
  this file follows the release notes for the `v1.1.0` smoke-test surface.
