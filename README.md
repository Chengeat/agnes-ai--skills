# Agnes AI Skills

OpenAI-compatible Python skills for **Agnes AI** multimodal models — image generation and video generation.

| Skill | Model | Capability |
|-------|-------|------------|
| `agnes-image-flash` | `agnes-image-2.0-flash` | Text-to-Image, Image-to-Image, Multi-Image Composition |
| `agnes-video-flash` | `agnes-video-2.5-flash` | Text-to-Video, Keyframe Control, Reference-Based Generation |

---

## Quick Start

### 1. Get API Key

Sign up at [platform.agnes-ai.com](https://platform.agnes-ai.com/login) → **Settings → API Keys** → Create a new key.

### 2. Set Environment Variable

```bash
export AGNES_API_KEY="your_api_key_here"
```

### 3. Install Dependencies

```bash
pip install httpx
```

---

## agnes-image-flash

Generate and edit images with **agnes-image-2.0-flash**.

### Supported Workflows

| Workflow |描述|
|----------|-------------|
| Text-to-Image | Generate images from text prompts |
| Image-to-Image | Edit / transform existing images |
| Multi-Image Composition | Combine multiple reference images |

### API Endpoint

```
POST https://apihub.agnes-ai.com/v1/images/generations
```

### Python Usage

```python
import httpx
import os

API_KEY = os.environ["AGNES_API_KEY"]
BASE_URL = "https://apihub.agnes-ai.com/v1"

async def generate_image(prompt: str, size: str = "1024x768", return_base64: bool = False) -> dict:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    payload = {
        "model": "agnes-image-2.0-flash",
        "prompt": prompt,
        "size": size,
    }
    if return_base64:
        payload["return_base64"] = True
    else:
        payload["extra_body"] = {"response_format": "url"}

    async with httpx.AsyncClient() as client:
        resp = await client.post(f"{BASE_URL}/images/generations", json=payload, headers=headers, timeout=360)
    resp.raise_for_status()
    return resp.json()


# Example: text-to-image
result = await generate_image(
    prompt="A clean product photo of a glass cube on a white studio background, soft shadows, high detail",
    size="1024x768",
)
print(result["data"][0]["url"])
```

### Image-to-Image Example

```python
result = await generate_image(
    prompt="Transform this into a cinematic cyberpunk style",
    size="1024x768",
    extra_body={"image": ["https://example.com/input.png"], "response_format": "url"},
)
```

### Available Sizes

| Size | Orientation |
|------|-------------|
| `1024x768` | Landscape |
| `1024x1024` | Square |
| `768x1024` | Portrait |

### Response Format

```json
{
  "created": 1780000000,
  "data": [{
    "url": "https://storage.googleapis.com/agnes-aigc/xxx.png",
    "b64_json": null,
    "revised_prompt": null
  }]
}
```

---

## agnes-video-flash

Generate videos with **agnes-video-2.5-flash** (asynchronous API).

### Supported Modes

| Mode |描述|
|------|-------------|
| `text` | Text-to-video |
| `keyframe` | First-frame / last-frame controlled generation |
| `reference` | Image or audio reference generation |

### API Endpoints

```
POST  https://apihub.agnes-ai.com/v1/videos            # create task
GET   https://apihub.agnes-ai.com/agnesapi?video_id=... # retrieve result
```

### Python Usage

```python
import httpx
import asyncio
import os

API_KEY = os.environ["AGNES_API_KEY"]
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
    """Create a video generation task. Returns video_id."""
    headers = {"Authorization": f"Bearer {API_KEY}", "Content-Type": "application/json"}
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
    return resp.json()["video_id"]


async def retrieve_video(video_id: str) -> dict:
    """Poll until completed or failed, then return result."""
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
            await asyncio.sleep(2)


# Example: text-to-video
video_id = await create_video_task(
    prompt="A silver sports car moves through a rain-soaked futuristic city street, neon reflections",
    seconds="5",
    mode="text",
    aspect_ratio="16:9",
)
result = await retrieve_video(video_id)
print(result["url"])
```

### Aspect Ratio → Output Resolution

| aspect_ratio | Output |
|--------------|--------|
| `21:9` | `1680x720` |
| `16:9` | `1280x704` |
| `4:3` | `960x720` |
| `1:1` | `720x720` |
| `3:4` | `720x960` |
| `9:16` | `720x1280` |

### Flash-Specific Limits

| Constraint | Limit |
|------------|-------|
| `size` | Only `"720P"` |
| Reference images | At most 5 |
| Reference audios | At most 3 |
| Video input | Not supported |

---

## Full API Reference

- [Agnes Image 2.0 Flash Docs](https://wiki.agnes-ai.com/en/docs/agnes-image-20-flash.md)
- [Agnes Video 2.5 Flash Docs](https://wiki.agnes-ai.com/en/docs/agnes-video-25-flash.md)
- [Agnes AI Overview](https://agnes-ai.com/doc/overview)

---

## License

MIT
