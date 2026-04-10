---
name: thankyou-api
description: Guide for using the ThankYou Generation API — authentication, submitting image/audio/video generation tasks, polling or receiving webhook callbacks, uploading reference files, and handling errors. Use nano-banana (image) and fish-audio (TTS) as reference examples.
user-invokable: true
args:
  - name: topic
    description: Specific area to focus on (auth, generate, poll, webhook, files, errors, models) — optional
    required: false
---

You are a developer assistant helping someone integrate with the ThankYou Generation API.
The base URL is `https://api.thankyouai.com/open/v1` (local dev: `http://localhost:8000/open/v1`).

## Overview

The ThankYou API is an async generation platform. The core flow is:

```
POST /generate          → returns { id, status: "queued" }
GET  /generations/{id}  → poll until status = "succeeded" | "failed"
Webhook callback        → optional push when the task reaches a subscribed terminal event
```

All requests require:
- `Authorization: Bearer <api_key>`
- `x-workspace-id: <workspace_id>` header — optional if the API key was created inside a workspace (key carries a default workspace); required only when overriding or if the key has no default workspace bound

---

## 1. Authentication

```bash
export TY_KEY="tk_YOUR_API_KEY"
export TY_WS="YOUR_WORKSPACE_ID"   # optional if key has a default workspace
export TY_BASE="https://api.thankyouai.com/open/v1"
```

---

## 2. Discover Models

```bash
# List all models
curl "$TY_BASE/models" -H "Authorization: Bearer $TY_KEY"

# Get field schema for a specific model
curl "$TY_BASE/models/detail?model_id=google/nano-banana/text-to-image" \
  -H "Authorization: Bearer $TY_KEY"
```

Response includes `primary_fields` (shown by default) and `advanced_fields` with
`key`, `type`, `description`, `required`, `default`, and `options`.

---

## 3. Submit a Generation — `POST /generate`

### Image: nano-banana series

**nano-banana (standard text-to-image)** — fast, basic parameter set
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/text-to-image",
    "input": {
      "prompt": "A serene mountain lake at golden hour, photorealistic",
      "aspect_ratio": "16:9"
    }
  }'
```
> Standard nano-banana only supports `prompt`, `aspect_ratio` (10 ratios), and `reference_assets`. It does **not** accept `image_size`, `num_images`, `seed`, or other advanced params.

**nano-banana/pro (premium with grounding)** — full parameter set
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/pro/text-to-image",
    "input": {
      "prompt": "The Eiffel Tower at sunset, photorealistic",
      "aspect_ratio": "3:2",
      "image_size": "2K",
      "num_images": 2,
      "seed": 777,
      "output_format": "png",
      "enable_web_search": true,
      "safety_tolerance": "3"
    }
  }'
```

**nano-banana/v2 (best quality, supports thinking)** — full parameter set
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/v2/text-to-image",
    "input": {
      "prompt": "A majestic phoenix rising from ancient ruins, fantasy art",
      "aspect_ratio": "16:9",
      "image_size": "2K",
      "num_images": 2,
      "seed": 12345,
      "output_format": "png",
      "enable_web_search": false,
      "thinking_level": "high",
      "safety_tolerance": "3"
    }
  }'
```

**nano-banana image editing (auto-routed when reference_assets provided)**

When `reference_assets` is included, the API automatically routes to the
`google/nano-banana/edit` model (image-to-image mode):
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/text-to-image",
    "input": {
      "prompt": "Transform into a painterly autumn scene",
      "aspect_ratio": "1:1",
      "reference_assets": [
        { "role": "reference", "url": "https://example.com/input.jpg" }
      ]
    }
  }'
```
> Reference image must be a **publicly accessible URL** (≥ a few KB). Use the
> Files endpoint (section 6) to upload your own images.

---

### Audio: fish-audio TTS

**Basic TTS**
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "fish-audio/text-to-speech",
    "input": {
      "text": "Welcome to ThankYou API.",
      "voice_id": "<voice_id>",
      "format": "mp3"
    }
  }'
```

**Advanced TTS (all parameters)**
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "fish-audio/text-to-speech",
    "input": {
      "text": "Testing advanced voice parameters.",
      "voice_id": "<voice_id>",
      "format": "wav",
      "temperature": 0.7,
      "top_p": 0.9,
      "latency": "normal",
      "normalize": true
    }
  }'
```

**Fish Audio field reference:**
| Field | Type | Description |
|-------|------|-------------|
| `text` | string | Text to synthesize |
| `voice_id` | string | Voice model ID — browse via `GET /voices` |
| `format` | enum | `mp3` \| `wav` \| `ogg` (default: `mp3`) |
| `temperature` | float 0–1 | Expressiveness randomness |
| `top_p` | float 0–1 | Nucleus sampling threshold |
| `latency` | enum | `normal` \| `balanced` \| `fast` |
| `normalize` | bool | Normalize output volume |

**Browse voices:**
```bash
curl "$TY_BASE/voices" -H "Authorization: Bearer $TY_KEY"
curl "$TY_BASE/voices/default" -H "Authorization: Bearer $TY_KEY"
```

---

## 4. Poll for Results — `GET /generations/{id}`

```bash
GEN_ID="<generation_id_returned_from_POST_generate>"

curl "$TY_BASE/generations/$GEN_ID" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS"
```

**Response:**
```json
{
  "id": "<generation_id>",
  "status": "succeeded",
  "model": "google/nano-banana/text-to-image",
  "output": [
    { "url": "https://static.thankyouai.com/image-outputs/…/1.png", "mime_type": "image/png" }
  ],
  "usage": { "credits_consumed": 0.045, "currency": "points" },
  "progress": 1.0,
  "resolved_params": { "prompt": "…", "aspect_ratio": "16:9" },
  "created_at": "2026-04-08T02:00:43Z",
  "finished_at": "2026-04-08T02:01:17Z",
  "error": null
}
```

**Status values:** `queued` → `running` → `succeeded` | `failed` | `cancelled`

**Shell poll loop:**
```bash
while true; do
  STATUS=$(curl -s "$TY_BASE/generations/$GEN_ID" \
    -H "Authorization: Bearer $TY_KEY" \
    -H "x-workspace-id: $TY_WS" | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  echo "Status: $STATUS"
  [[ "$STATUS" == "succeeded" || "$STATUS" == "failed" ]] && break
  sleep 5
done
```

**List all past generations:**
```bash
curl "$TY_BASE/generations?page=1&page_size=20" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS"
```

---

## 5. Webhook Mode — push terminal events instead of polling

If you do not want to poll `GET /generations/{id}`, include a `webhook` object in
`POST /generate`. ThankYou will POST the terminal event to your HTTPS endpoint.
You can still poll as a fallback even when webhook delivery is enabled.

**Create a generation with webhook delivery:**
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/text-to-image",
    "input": {
      "prompt": "A cinematic fox running through neon rain",
      "aspect_ratio": "16:9"
    },
    "webhook": {
      "url": "https://example.com/callback",
      "events": ["generation.completed", "generation.failed"]
    }
  }'
```

**Webhook rules:**
- `webhook.url` must be **HTTPS**
- `webhook.events` may contain only:
  - `generation.completed`
  - `generation.failed`
  - `generation.cancelled`
- Webhooks are delivered only for terminal states, so they complement polling rather
  than replacing the initial `POST /generate` response

**Webhook request format:**
```http
POST https://example.com/callback
X-ThankYou-Signature: sha256=<hmac>
X-ThankYou-Timestamp: 1711526400
Content-Type: application/json

{
  "event": "generation.completed",
  "generation": {
    "id": "gen_abc123",
    "status": "succeeded",
    "model": "google/nano-banana/text-to-image",
    "output": [
      { "url": "https://static.thankyouai.com/...", "mime_type": "image/png" }
    ],
    "error": null
  }
}
```

**Signature verification:**
- Each API key has its own `webhook_secret`
- Signature format: `sha256=HMAC_SHA256(webhook_secret, "{timestamp}.{raw_body}")`
- Recommended replay protection: reject requests when `abs(now - timestamp) > 300s`

**Python verification example:**
```python
import hashlib
import hmac
import time

def verify_thankyou_webhook(raw_body: bytes, timestamp: str, signature: str, secret: str) -> bool:
    if abs(time.time() - int(timestamp)) > 300:
        return False
    signed = f"{timestamp}.".encode("utf-8") + raw_body
    expected = "sha256=" + hmac.new(secret.encode("utf-8"), signed, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**Delivery semantics:**
- Retry schedule: roughly **10s → 60s → 5min**
- If all retries fail, fetch the final state manually with `GET /generations/{id}`
- Your webhook handler should be idempotent and key off `generation.id`

---

## 6. Upload Reference Files — `POST /files`

Use the two-step presigned PUT flow to upload images for use as reference_assets:

**Step 1 — Create upload session:**
```bash
SESSION=$(curl -s -X POST "$TY_BASE/files" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"content_type\": \"image/jpeg\", \"size_bytes\": $(wc -c < my_image.jpg)}")

UPLOAD_URL=$(echo "$SESSION" | python3 -c "import sys,json; print(json.load(sys.stdin)['upload_url'])")
FILE_URL=$(echo "$SESSION" | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")
```

**Step 2 — PUT the file directly to the storage URL:**
```bash
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --data-binary @my_image.jpg
# → 200 OK
```

Now use `$FILE_URL` as the reference image URL in generation requests.

**Limits:** Max 50 MB per file. Upload URL expires in 15 minutes.

---

## 7. Error Handling

The API uses two error surfaces:

### HTTP errors (immediate rejection — no task created)
```json
{
  "error": {
    "code": "blocked",
    "type": "validation_error",
    "message": "Generation blocked: invalid_param:safety_tolerance",
    "retryable": false,
    "details": { "blocking_reasons": ["invalid_param:safety_tolerance"] }
  }
}
```

### Async task errors (in `GET /generations/{id}`)
```json
{
  "error": {
    "code": "provider_failed",
    "type": "provider_error",
    "message": "provider error: The AI provider rejected the request (invalid parameters or unsupported input format).",
    "retryable": false
  }
}
```

**Error type classification:**
| `type` | Message prefix | Meaning | Retryable? |
|--------|---------------|---------|------------|
| `api_error` | `openapi error:` | Billing, enqueue, or scheduling failure | Varies |
| `queue_error` | `openapi generate error:` | Internal queue service fault | Usually yes |
| `provider_error` | `provider error:` | Upstream AI model rejected or timed out | Varies |

**Common error codes:**
| `code` | `type` | Notes |
|--------|--------|-------|
| `billing_error` | api_error | Insufficient credits |
| `queue_enqueue_failed` | api_error | Retry the request |
| `queue_server_error` | queue_error | Transient — retry |
| `provider_failed` | provider_error | Check `message` for reason |
| `provider_error` | provider_error | Transient provider issue |

**Retry strategy:** Check `retryable: true` before retrying. For `provider_error`,
exponential backoff with 2–3 attempts is recommended.

---

## 8. Python Quick-Start

```python
import os
import time
import requests

BASE = "https://api.thankyouai.com/open/v1"

def _headers(workspace_id: str | None = None) -> dict:
    h = {
        "Authorization": f"Bearer {os.environ['TY_KEY']}",
        "Content-Type": "application/json",
    }
    # x-workspace-id is optional if the API key has a default workspace
    ws = workspace_id or os.environ.get("TY_WS")
    if ws:
        h["x-workspace-id"] = ws
    return h

def generate(model: str, input_params: dict, workspace_id: str | None = None) -> dict:
    r = requests.post(
        f"{BASE}/generate",
        json={"model": model, "input": input_params},
        headers=_headers(workspace_id),
    )
    r.raise_for_status()
    gen_id = r.json()["id"]

    for _ in range(60):  # poll up to 5 minutes
        time.sleep(5)
        result = requests.get(f"{BASE}/generations/{gen_id}", headers=_headers(workspace_id)).json()
        if result["status"] in ("succeeded", "failed", "cancelled"):
            if result["status"] != "succeeded":
                raise RuntimeError(result.get("error"))
            return result
    raise TimeoutError(f"Generation {gen_id} did not complete")

# Usage:
#   export TY_KEY="tk_..."
#   export TY_WS="..."   # optional

# Image
result = generate("google/nano-banana/text-to-image", {
    "prompt": "A serene mountain lake at golden hour",
    "aspect_ratio": "16:9",
})
print(result["output"][0]["url"])

# TTS
result = generate("fish-audio/text-to-speech", {
    "text": "Hello from ThankYou API.",
    "voice_id": "03397b4c4be74759b72533b663fbd001",
    "format": "mp3",
})
print(result["output"][0]["url"])
```

---

## Key Gotchas

- **`x-workspace-id` is optional if** the API key was created inside a workspace (the key carries a default `workspace_id`). Pass the header explicitly only when you need to override the default or the key has no workspace bound — missing both raises `missing_workspace` 422.
- **`reference_assets` auto-routes** to the edit/image-to-image model variant
- **Reference image URLs must be public** — upload via `POST /files` for private assets
- **`safety_tolerance` must be a string** (`"1"`–`"6"`), not an integer. Integer values are rejected as `invalid_param:safety_tolerance`. The schema `options` is a string array and the validator enforces the type strictly.
- **`thinking_level` is `nano-banana/v2` only.** Pro doesn't expose it — sending it on Pro raises `invalid_param:thinking_level`.
- **Empty-string `prompt` is not caught at validation.** It enqueues but fails at the provider with `provider_failed`. Always ensure `prompt.strip() != ""` client-side.
- **Standard `nano-banana` ≠ `pro`/`v2`.** The standard model has a smaller parameter set (no `image_size`, `num_images`, `seed`, `output_format`, etc.). Check `/models/detail` before assuming a field exists.
- **`/models/detail` may occasionally return a stripped-down schema** (empty `advanced_fields`) during deployments or cache misses. If fields you expect are missing, retry the detail request after a few seconds.
- **Output URLs are temporary** — download and store them; they may expire
- **Poll at 3–10s intervals** — most image tasks complete in 15–60s, TTS in 5–15s
- **Webhook signatures must use the raw request body** — do not JSON re-serialize before verifying `X-ThankYou-Signature`
- **Webhook handlers should be idempotent** — retries can deliver the same terminal event more than once
