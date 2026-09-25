---
name: "agnes-image-flash"
description: "Generate and edit images using the agnes-image-2.0-flash model via Agnes AI API. Invoke when user asks to create images, edit photos, generate artwork, or create visual content from text prompts."
---

# Agnes Image 2.0 Flash

Generate and edit images using the **agnes-image-2.0-flash** model through the Agnes AI API.

## API Information

- **Base URL**: `https://apihub.agnes-ai.com/v1`
- **Endpoint**: `POST /v1/images/generations`
- **Model**: `agnes-image-2.0-flash`
- **Authentication**: Bearer token in `Authorization` header

## Supported Workflows

| Workflow | Description | Required Fields |
|----------|-------------|-----------------|
| Text-to-image | Generate image from text prompt | `model`, `prompt`, `size` |
| Image-to-image | Edit/transform existing images | `model`, `prompt`, `size`, `extra_body.image` |
| Multi-image composition | Combine multiple reference images | `model`, `prompt`, `size`, `extra_body.image` |

## Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `model` | string | Yes | Model name: `agnes-image-2.0-flash` |
| `prompt` | string | Yes | Text description of the target image or editing instruction |
| `size` | string | Yes | Output image size, e.g. `1024x768`, `1024x1024`, `768x1024` |
| `image` | string\[\] | No (img2img) | Input image array — public URLs or Data URI Base64 |
| `return_base64` | boolean | No | Return result as Base64 instead of URL |
| `extra_body.response_format` | string | No | Output format: `url` or `b64_json` |

> **Important**: `response_format` must be inside `extra_body`, NOT at the top level.

## Output Sizes

- `1024x768` — landscape
- `1024x1024` — square
- `768x1024` — portrait

## Response Format (URL output)

```json
{
  "created": 1780000000,
  "data": [
    {
      "url": "https://storage.googleapis.com/agnes-aigc/xxx.png",
      "b64_json": null,
      "revised_prompt": null
    }
  ]
}
```

## Response Format (Base64 output)

```json
{
  "created": 1780000000,
  "data": [
    {
      "url": null,
      "b64_json": "iVBORw0KGgoAAAANSUhEUgAA...",
      "revised_prompt": null
    }
  ]
}
```

## Usage Examples

### Text-to-Image (URL output)

```bash
curl https://apihub.agnes-ai.com/v1/images/generations \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-image-2.0-flash",
    "prompt": "A clean product photo of a glass cube on a white studio background, soft shadows, high detail",
    "size": "1024x768",
    "extra_body": {
      "response_format": "url"
    }
  }'
```

### Text-to-Image (Base64 output)

```bash
curl https://apihub.agnes-ai.com/v1/images/generations \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-image-2.0-flash",
    "prompt": "A clean product photo of a glass cube on a white studio background, soft shadows, high detail",
    "size": "1024x768",
    "return_base64": true
  }'
```

### Image-to-Image

```bash
curl https://apihub.agnes-ai.com/v1/images/generations \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-image-2.0-flash",
    "prompt": "Transform this image into a cinematic cyberpunk style while preserving the main subject and composition",
    "size": "1024x768",
    "extra_body": {
      "image": ["https://example.com/input-image.png"],
      "response_format": "url"
    }
  }'
```

### Multi-Image Composition

```bash
curl https://apihub.agnes-ai.com/v1/images/generations \
  -H "Authorization: Bearer $AGNES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agnes-image-2.0-flash",
    "prompt": "Combine the two characters into an intense fantasy battle scene, dynamic lighting, detailed background",
    "size": "1024x768",
    "extra_body": {
      "image": [
        "https://example.com/character-1.png",
        "https://example.com/character-2.png"
      ],
      "response_format": "url"
    }
  }'
```

## Python Example

```python
import httpx
import os

API_KEY = os.environ.get("AGNES_API_KEY")
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
```

## Prompt Best Practices

**Text-to-Image prompt structure:**
```
[Subject] + [Scene/Background] + [Style] + [Lighting] + [Composition] + [Quality Requirements]
```

**Image Editing prompt structure:**
```
[Editing Instruction] + [Elements to Preserve] + [Target Style/Scene] + [Lighting] + [Quality Requirements]
```

**Example:**
> A professional product photo of a wireless headphone on a clean white background, soft studio lighting, sharp details, commercial photography style

## Important Notes

- Client timeout should be set to **60s–360s** — image generation can take several seconds
- For image-to-image, input image URLs must be publicly accessible
- Use Data URI Base64 if the source image cannot be made public
- The current price for all tiers is **free** (promotional period)
