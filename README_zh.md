> [English](./README.md) | 简体中文
# Agnes AI 技能库
适用于 Agnes AI 多模态模型，兼容 OpenAI 接口规范的 Python 能力库 —— 图像生成与视频生成

| 技能 | 模型 | 能力 |
|-------|-------|------------|
| `agnes-image-flash` | `agnes-image-2.0-flash` | 文生图、图生图、多图融合合成 |
| `agnes-video-flash` | `agnes-video-2.5-flash` | 文生视频、关键帧控制、基于参考素材生成 |

---

## 快速上手

### 1. 获取 API Key
前往平台注册：[platform.agnes-ai.com](https://platform.agnes-ai.com/login) → **设置 → API 密钥** → 创建新密钥

### 2. 设置环境变量
```bash
export AGNES_API_KEY="在此填入你的API密钥"
```

### 3. 安装依赖
```bash
pip install httpx
```

---

## agnes-image-flash
使用 `agnes-image-2.0-flash` 生成与编辑图像

### 支持的工作流
| 工作流 | 描述 |
|----------|-------------|
| Text-to-Image（文生图） | 根据文本提示词生成图片 |
| Image-to-Image（图生图） | 编辑/转换已有图片 |
| Multi-Image Composition（多图合成） | 融合多张参考图进行创作 |

### API 接口地址
```
POST [https://apihub.agnes-ai.com/v1/images/generations](https://apihub.agnes-ai.com/v1/images/generations)
```

### Python 使用示例
```python
import httpx
import os

API_KEY = os.environ["AGNES_API_KEY"]
BASE_URL = "[https://apihub.agnes-ai.com/v1](https://apihub.agnes-ai.com/v1)"

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


# 示例：文生图
result = await generate_image(
    prompt="白色摄影棚背景中的玻璃立方体产品照，柔和阴影，高细节",
    size="1024x768",
)
print(result["data"][0]["url"])
```

### 图生图示例
```python
result = await generate_image(
    prompt="将画面转换为赛博朋克电影风格",
    size="1024x768",
    extra_body={"image": ["[https://example.com/input.png](https://example.com/input.png)"], "response_format": "url"},
)
```

### 可用尺寸
| 尺寸 | 方向 |
|------|-------------|
| `1024x768` | 横向（风景） |
| `1024x1024` | 正方形 |
| `768x1024` | 纵向（人像） |

### 返回数据格式
```json
{
  "created": 1780000000,
  "data": [{
    "url": "[https://storage.googleapis.com/agnes-aigc/xxx.png](https://storage.googleapis.com/agnes-aigc/xxx.png)",
    "b64_json": null,
    "revised_prompt": null
  }]
}
```

---

## agnes-video-flash
使用 `agnes-video-2.5-flash` 生成视频（异步 API）

### 支持模式
| 模式 | 描述 |
|------|-------------|
| `text` | 文生视频 |
| `keyframe` | 首帧/末帧控制生成 |
| `reference` | 基于图片或音频参考生成 |

### API 接口地址
```
POST  [https://apihub.agnes-ai.com/v1/videos](https://apihub.agnes-ai.com/v1/videos)            # 创建任务
GET   [https://apihub.agnes-ai.com/agnesapi?video_id=](https://apihub.agnes-ai.com/agnesapi?video_id=)... # 查询并获取结果
```

### Python 使用示例
```python
import httpx
import asyncio
import os

API_KEY = os.environ["AGNES_API_KEY"]
BASE_URL = "[https://apihub.agnes-ai.com/v1](https://apihub.agnes-ai.com/v1)"
RETRIEVE_URL = "[https://apihub.agnes-ai.com/agnesapi](https://apihub.agnes-ai.com/agnesapi)"


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
    """创建视频生成任务，返回video_id任务ID"""
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
    """循环轮询，直到任务完成或失败，返回结果"""
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


# 示例：文生视频
video_id = await create_video_task(
    prompt="银色跑车行驶在雨后未来都市街道，霓虹灯光在地面反光",
    seconds="5",
    mode="text",
    aspect_ratio="16:9",
)
result = await retrieve_video(video_id)
print(result["url"])
```

### 宽高比与输出分辨率对照表
| 宽高比 | 输出分辨率 |
|--------------|--------|
| `21:9` | `1680x720` |
| `16:9` | `1280x704` |
| `4:3` | `960x720` |
| `1:1` | `720x720` |
| `3:4` | `720x960` |
| `9:16` | `720x1280` |

### Flash 版本限制
| 约束项 | 限制 |
|------------|-------|
| `size` | 仅支持 `"720P"` |
| 参考图片 | 最多5张 |
| 参考音频 | 最多3条 |
| 视频输入 | 暂不支持 |

---

## 完整API参考文档
- [https://wiki.agnes-ai.com/en/docs/agnes-image-20-flash.md](https://wiki.agnes-ai.com/en/docs/agnes-image-20-flash.md)
- [https://wiki.agnes-ai.com/en/docs/agnes-video-25-flash.md](https://wiki.agnes-ai.com/en/docs/agnes-video-25-flash.md)
- [https://agnes-ai.com/doc/overview](https://agnes-ai.com/doc/overview)

---

## 许可证
MIT
