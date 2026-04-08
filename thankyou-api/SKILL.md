---
name: thankyou-api
description: Guide for using the ThankYou Generation API — authentication, submitting image/audio/video generation tasks, polling results, uploading reference files, and handling errors. Use nano-banana (image) and fish-audio (TTS) as reference examples.
user-invokable: true
args:
  - name: topic
    description: Specific area to focus on (auth, generate, poll, files, errors, models) — optional
    required: false
---

You are a developer assistant helping someone integrate with the ThankYou Generation API.
The base URL is `https://api.thankyouai.com/open/v1` (local dev: `http://localhost:8000/open/v1`).

## Overview

The ThankYou API is an async generation platform. The core flow is:

```
POST /generate          → returns { id, status: "queued" }
GET  /generations/{id}  → poll until status = "succeeded" | "failed"
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

**nano-banana (standard text-to-image)**
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/text-to-image",
    "input": {
      "prompt": "A serene mountain lake at golden hour, photorealistic",
      "aspect_ratio": "16:9",
      "num_images": 1
    }
  }'
```

**nano-banana/pro (higher quality, supports seed)**
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/pro/text-to-image",
    "input": {
      "prompt": "Futuristic cityscape at night with neon lights",
      "image_size": "1024x1024",
      "seed": 42
    }
  }'
```

**nano-banana/v2 (latest generation, supports seed)**
```bash
curl -X POST "$TY_BASE/generate" \
  -H "Authorization: Bearer $TY_KEY" \
  -H "x-workspace-id: $TY_WS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana/v2/text-to-image",
    "input": {
      "prompt": "Wise elderly scientist in a cozy laboratory",
      "seed": 123
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
> Files endpoint (section 5) to upload your own images.

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

## 5. Upload Reference Files — `POST /files`

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

## 6. Error Handling

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

## 7. Python Quick-Start

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
- **`safety_tolerance` / `thinking_level`** may appear in model detail for some models but are not supported — omit them
- **Output URLs are temporary** — download and store them; they may expire
- **Poll at 3–10s intervals** — most image tasks complete in 15–60s, TTS in 5–15s
