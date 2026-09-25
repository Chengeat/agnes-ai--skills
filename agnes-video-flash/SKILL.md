---
name: "agnes-video-flash"
description: "Generate videos using the agnes-video-2.5-flash model via Agnes AI API. Invoke when user asks to create videos, generate motion clips, animate images, or produce video content from text prompts."
---

# Agnes Video 2.5 Flash

Generate videos using the **agnes-video-2.5-flash** model through the Agnes AI API.

## API Information

- **Base URL**: `https://apihub.agnes-ai.com/v1`
- **Create Task**: `POST /v1/videos`
- **Retrieve Task**: `GET /agnesapi?video_id=<VIDEO_ID>&model_name=agnes-video-2.5-flash`
- **Model**: `agnes-video-2.5-flash`
- **Authentication**: Bearer token in `Authorization` header

## Supported Workflows

| Mode | Description | Required Fields |
|------|-------------|-----------------|
| `text` | Text-to-video generation | `model`, `prompt`, `mode` |
| `keyframe` | First-frame / last-frame controlled generation | `model`, `prompt`, `mode`, at least one of `first_frame` or `last_frame` |
| `reference` | Image/audio reference generation | `model`, `prompt`, `mode`, at least one of `images` or `audios` |

## Common Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `model` | string | Yes | `agnes-video-2.5-flash` |
| `prompt` | string | Yes | Video description. In reference mode, use `<Picture N>` and `<Audio N>` for inputs. |
| `mode` | string | Yes | `text`, `keyframe`, or `reference` |
| `seconds` | string | No | Duration as string from `"4"` to `"12"`. Default: `"5"` |
| `size` | string | No | Must be `"720P"` for Flash. Other values return HTTP 400 |
| `aspect_ratio` | string | No | Default `16:9`. Options: `21:9`, `16:9`, `4:3`, `1:1`, `3:4`, `9:16` |
| `seed` | integer | No | Random seed for reproducible results |
| `n` | integer | No | Number of videos; only `1` supported, default `1` |

## Aspect Ratio → Output Resolution

| aspect_ratio | Output pixels |
|--------------|---------------|
| `21:9` | `1680x720` |
| `16:9` | `1280x704` |
| `4:3` | `960x720` |
| `1:1` | `720x720` |
| `3:4` | `720x960` |
| `9:16` | `720x1280` |

## Flash-Specific Limits

| Constraint | Limit | Error if exceeded |
|------------|-------|--------------------|
| `size` | Only `"720P"` | HTTP 400: `size must be 720P` |
| `images` count | At most 5 | HTTP 400: `images length must not exceed 5` |
| `audios` count | At most 3 | HTTP 400: `audios length must not exceed 3` |
| `videos` input | Not supported | HTTP 400: `videos is not supported` |

## Mode Rules

| Mode | Use case | Required media | Disallowed media |
|------|----------|----------------|------------------|
| `text` | Text-to-video | None | `first_frame`, `last_frame`, `images`, `audios`, `videos` |
| `keyframe` | Frame control | At least one of `first_frame` or `last_frame` | `images`, `audios`, `videos` |
| `reference` | Reference-based generation | At least one of `images` or `audios` | `first_frame`, `last_frame`, `videos` |

## Step 1: Create Task

```bash
curl -sS -X POST "https://apihub.agnes-ai.com/v1/videos" \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-video-2.5-flash",
    "prompt": "A silver sports car moves slowly through a rain-soaked futuristic city street, neon reflections, cinematic camera motion",
    "seconds": "5",
    "mode": "text",
    "size": "720P",
    "aspect_ratio": "16:9"
  }'
```

Response (save `video_id`):
```json
{
  "id": "task_uuid",
  "task_id": "task_uuid",
  "video_id": "vid_xxxxx",
  "status": "submitted"
}
```

## Step 2: Retrieve Result

Poll every **1–2 seconds** until `status` is `completed` or `failed`:

```bash
curl -sS "https://apihub.agnes-ai.com/agnesapi?video_id=VIDEO_ID&model_name=agnes-video-2.5-flash" \
  -H "Authorization: Bearer $AGNES_API_KEY"
```

## Python Example

```python
import httpx
import time
import os

API_KEY = os.environ.get("AGNES_API_KEY")
BASE_URL = "https://apihub.agnes-ai.com/v1"
RETRIEVE_URL = "https://apihub.agnes-ai.com/agnesapi"


async def create_video_task(
    prompt: str,
    seconds: str = "5",
    mode: str = "text",
    aspect_ratio: str = "16:9",
    first_frame: str = None,
    last_frame: str = None,
    images: list = None,
    audios: list = None,
) -> str:
    """Create a video generation task and return the video_id."""
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    payload = {
        "model": "agnes-video-2.5-flash",
        "prompt": prompt,
        "seconds": seconds,
        "mode": mode,
        "size": "720P",
        "aspect_ratio": aspect_ratio,
    }
    if first_frame:
        payload["first_frame"] = first_frame
    if last_frame:
        payload["last_frame"] = last_frame
    if images:
        payload["images"] = images
    if audios:
        payload["audios"] = audios

    async with httpx.AsyncClient() as client:
        resp = await client.post(f"{BASE_URL}/videos", json=payload, headers=headers, timeout=60)
    resp.raise_for_status()
    data = resp.json()
    return data["video_id"]


async def retrieve_video(video_id: str) -> dict:
    """Poll until video is completed or failed, then return the result."""
    params = {"video_id": video_id, "model_name": "agnes-video-2.5-flash"}
    headers = {"Authorization": f"Bearer {API_KEY}"}

    async with httpx.AsyncClient() as client:
        while True:
            resp = await client.get(RETRIEVE_URL, params=params, headers=headers, timeout=30)
            resp.raise_for_status()
            result = resp.json()
            status = result.get("status")
            if status in ("completed", "failed"):
                return result
            await asyncio.sleep(2)  # poll every 2 seconds
```

## Request Examples

### Text-to-Video

```bash
curl -sS -X POST "$AGNES_BASE_URL/videos" \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-video-2.5-flash",
    "prompt": "Three cats form a tiny brass band and march through a moonlit forest, smooth backward tracking shot",
    "seconds": "5",
    "mode": "text",
    "size": "720P",
    "aspect_ratio": "16:9"
  }'
```

### Keyframe Control

```bash
curl -sS -X POST "$AGNES_BASE_URL/videos" \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-video-2.5-flash",
    "prompt": "The character turns naturally and walks toward the window as the camera slowly pushes in",
    "seconds": "5",
    "mode": "keyframe",
    "size": "720P",
    "first_frame": "https://example.com/first.png",
    "last_frame": "https://example.com/last.png"
  }'
```

### Image Reference

```bash
curl -sS -X POST "$AGNES_BASE_URL/videos" \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-video-2.5-flash",
    "prompt": "Use the character and art style in <Picture 1> as reference. The character runs through a flower field while preserving appearance",
    "seconds": "5",
    "mode": "reference",
    "size": "720P",
    "aspect_ratio": "16:9",
    "images": ["https://example.com/character.png"]
  }'
```

## Important Notes

- All media URLs must be publicly accessible and remain valid until task completion
- The API key must be kept secure — never expose in client code or public repositories
- Retrieval must include `model_name=agnes-video-2.5-flash` for `keyframe` and `reference` modes
- Task retrieval without `model_name` only works for `text` mode tasks
- Current price: **free** (promotional period)
