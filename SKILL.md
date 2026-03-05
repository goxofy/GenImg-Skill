---
name: freebanana-api
description: Generate images via freebanana.pro API (text-to-image and image-to-image) with a tested anti-block pattern.
homepage: https://freebanana.pro/
---

# FreeBanana API Skill

Purpose
- Call freebanana.pro API directly for image generation.
- Supports:
  - Text-to-image (`inputImages: []`)
  - Image-to-image (`inputImages: ["data:image/...;base64,..."]`)

Verified endpoints
- `POST /api/generate` submit a task
- `GET /api/task/{taskId}` query status

----------------------------------------

## Execution Policy (Important)

Default execution mode for all Python examples in this skill:
- Prefer `uv run`.
- Prefer inline script dependencies in the `.py` header.

Use this format at the top of scripts:

```python
# /// script
# requires-python = ">=3.9"
# dependencies = [
#   "curl_cffi>=0.13.0",
# ]
# ///
```

Run with:

```bash
uv run your_script.py
```

----------------------------------------

## Proven Success Pattern

This API is protected by probabilistic anti-bot checks.

Most reliable pattern:
1) Use `curl_cffi`
2) Use `requests.Session(impersonate="chrome136")`
3) Warm up first: `GET https://freebanana.pro/`
4) Keep headers close to browser capture
5) Use random 32-char hex `visitorId`
6) Retry submit with exponential backoff

----------------------------------------

## Request fields

- `prompt`: string
- `inputImages`: string[]
  - text-to-image: `[]`
  - image-to-image: `["data:image/jpeg;base64,..."]`
- `aspectRatio`: `landscape` | `portrait`
- `timestamp`: number
- `visitorId`: string (recommended random 32-char hex)

----------------------------------------

## timestamp algorithm

1. `sec = floor(nowMs / 1000)`
2. `raw = visitorId + sec + prompt`
3. JS-style 32-bit hash: `h = (h<<5) - h + charCode`
4. `checksum = abs(h % 100)` padded to 2 digits
5. `timestamp = floor(nowMs / 100) * 100 + checksum`

----------------------------------------

## Recommended headers

- `accept: */*`
- `accept-language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7`
- `content-type: application/json`
- `origin: https://freebanana.pro`
- `referer: https://freebanana.pro/`
- `priority: u=1, i`
- `sec-ch-ua: "Not:A-Brand";v="99", "Google Chrome";v="145", "Chromium";v="145"`
- `sec-ch-ua-mobile: ?0`
- `sec-ch-ua-platform: "macOS"`
- `sec-fetch-dest: empty`
- `sec-fetch-mode: cors`
- `sec-fetch-site: same-origin`
- `user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36`

Stability tip:
- Match frontend body style:
  - `body = json.dumps(payload, ensure_ascii=False).replace('{', '{ ', 1)`

----------------------------------------

## Minimal template (uv + curl_cffi)

```python
# /// script
# requires-python = ">=3.9"
# dependencies = [
#   "curl_cffi>=0.13.0",
# ]
# ///

import base64
import json
import secrets
import time
from curl_cffi import requests


def js_hash(s: str) -> int:
    h = 0
    for ch in s:
        h = ((h << 5) - h + ord(ch)) & 0xffffffff
    if h & 0x80000000:
        h = -((~h + 1) & 0xffffffff)
    return h


def make_timestamp(visitor_id: str, prompt: str) -> int:
    now = int(time.time() * 1000)
    sec = str(now // 1000)
    h = js_hash(visitor_id + sec + prompt)
    checksum = int(str(abs(h % 100)).zfill(2))
    return (now // 100) * 100 + checksum


def build_headers():
    return {
        "accept": "*/*",
        "accept-language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7",
        "content-type": "application/json",
        "origin": "https://freebanana.pro",
        "priority": "u=1, i",
        "referer": "https://freebanana.pro/",
        "sec-ch-ua": '"Not:A-Brand";v="99", "Google Chrome";v="145", "Chromium";v="145"',
        "sec-ch-ua-mobile": "?0",
        "sec-ch-ua-platform": '"macOS"',
        "sec-fetch-dest": "empty",
        "sec-fetch-mode": "cors",
        "sec-fetch-site": "same-origin",
        "user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36",
    }


def run_generate(prompt: str, input_images: list[str], aspect_ratio: str = "portrait"):
    sess = requests.Session(impersonate="chrome136")
    headers = build_headers()

    # warmup
    sess.get("https://freebanana.pro/", headers=headers, timeout=30)

    visitor_id = secrets.token_hex(16)
    payload = {
        "prompt": prompt,
        "inputImages": input_images,
        "aspectRatio": aspect_ratio,
        "timestamp": make_timestamp(visitor_id, prompt),
        "visitorId": visitor_id,
    }

    body = json.dumps(payload, ensure_ascii=False).replace("{", "{ ", 1)
    r = sess.post("https://freebanana.pro/api/generate", data=body.encode("utf-8"), headers=headers, timeout=60)
    if r.status_code != 202:
        raise RuntimeError(f"submit failed: {r.status_code} {r.text[:200]}")

    task_id = r.json()["taskId"]

    for _ in range(90):
        time.sleep(5)
        rr = sess.get(f"https://freebanana.pro/api/task/{task_id}", headers=headers, timeout=60)
        rr.raise_for_status()
        data = rr.json()
        status = data.get("status")
        if status == "completed" and data.get("result", {}).get("imageUrl"):
            image_url = data["result"]["imageUrl"]
            img = sess.get(image_url, timeout=120)
            img.raise_for_status()
            return img.content, image_url
        if status == "failed":
            raise RuntimeError(data)

    raise TimeoutError("poll timeout")


# text-to-image example
img_bytes, image_url = run_generate("dark fantasy city night", input_images=[])
open("text2img.jpg", "wb").write(img_bytes)

# image-to-image example
raw = open("input.jpg", "rb").read()
data_url = "data:image/jpeg;base64," + base64.b64encode(raw).decode("ascii")
img_bytes, image_url = run_generate(
    "convert this image into a dark cinematic style, cold desaturated palette, high contrast",
    input_images=[data_url],
)
open("img2img.jpg", "wb").write(img_bytes)
```

----------------------------------------

## Retry recommendation

Use up to 5 retries for submit step.
Suggested delays: `2s, 5s, 12s, 20s, 30s` (+ random jitter).

Common failures:
- `403 {"error":"errors.request.failed"}`
  - anti-bot block; retry with backoff
- `status=failed` with upstream no-response/token errors
  - upstream temporarily unavailable; retry later

----------------------------------------

## Safety

- Only generate user-authorized content.
- Avoid high-frequency concurrent requests.