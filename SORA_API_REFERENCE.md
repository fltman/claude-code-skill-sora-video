# OpenAI Sora Video Generation API -- Complete Reference

Last updated: 2026-02-21

---

## Table of Contents

1. [Overview](#1-overview)
2. [Authentication](#2-authentication)
3. [API Endpoints](#3-api-endpoints)
4. [Models](#4-models)
5. [Parameters Reference](#5-parameters-reference)
6. [Async Workflow](#6-async-workflow-create-poll-download)
7. [Python SDK Usage](#7-python-sdk-usage)
8. [REST API (curl) Usage](#8-rest-api-curl-usage)
9. [Webhook Notifications](#9-webhook-notifications)
10. [Response Objects](#10-response-objects)
11. [Pricing](#11-pricing)
12. [Rate Limits](#12-rate-limits)
13. [Content Restrictions](#13-content-restrictions)
14. [Error Handling](#14-error-handling)
15. [Azure OpenAI Variant](#15-azure-openai-variant)

---

## 1. Overview

Sora is OpenAI's video generation model capable of creating richly detailed, dynamic video clips
with synchronized audio from natural language prompts or reference images. The API uses an
asynchronous job-based workflow: you submit a generation request, receive a job ID, poll for
completion (or use webhooks), and then download the resulting MP4.

Key capabilities:
- Text-to-video generation
- Image-to-video generation (reference image as first frame)
- Video remixing (modify an existing generated video)
- Synchronized audio generation
- Multiple resolution and duration options

---

## 2. Authentication

### OpenAI Direct API

Standard Bearer token authentication using your OpenAI API key:

```
Authorization: Bearer sk-your-api-key-here
```

Environment variable:
```bash
export OPENAI_API_KEY="sk-your-api-key-here"
```

### Access Requirements

- API access requires an OpenAI account with billing enabled
- As of January 2026, free-tier users cannot generate videos
- ChatGPT Plus ($20/month) or Pro ($200/month) subscribers have access
- API tier determines rate limits (Tier 2 through Enterprise)

---

## 3. API Endpoints

All endpoints are relative to `https://api.openai.com/v1`.

| Method | Endpoint                          | Purpose                              |
|--------|-----------------------------------|--------------------------------------|
| POST   | `/videos`                         | Create a new video generation job    |
| GET    | `/videos/{video_id}`              | Retrieve job status and metadata     |
| GET    | `/videos/{video_id}/content`      | Download completed video/thumbnail   |
| GET    | `/videos`                         | List all videos (paginated)          |
| DELETE | `/videos/{video_id}`              | Delete a video                       |
| POST   | `/videos/{video_id}/remix`        | Create a variation of existing video |

---

## 4. Models

### sora-2 (Standard)

- Model ID: `sora-2`
- Alias/snapshot: `sora-2-2025-10-06`, `sora-2-2025-12-08`
- Designed for speed and flexibility
- Good quality results, fast generation
- Ideal for rapid iteration, prototyping, social media content
- Supported resolutions: 720x1280, 1280x720
- Supported durations: 4s, 8s, 12s
- Typical generation time: 30-90 seconds
- Cost: $0.10/second of generated video

### sora-2-pro (Professional)

- Model ID: `sora-2-pro`
- Alias/snapshot: `sora-2-pro-2025-10-06`
- Higher quality output, production-grade
- Enhanced rendering, smoother motion, better visual coherence
- Supported resolutions: 720x1280, 1280x720, 1024x1792, 1792x1024
- Supported durations: 4s, 8s, 12s
- Typical generation time: 1-5 minutes
- Cost: $0.30/second (720p), $0.50/second (1024p)

### All Valid Model Strings

```
"sora-2"
"sora-2-pro"
"sora-2-2025-10-06"
"sora-2-pro-2025-10-06"
"sora-2-2025-12-08"
```

---

## 5. Parameters Reference

### POST /videos -- Create Video

| Parameter         | Type   | Required | Default    | Description                                      |
|-------------------|--------|----------|------------|--------------------------------------------------|
| `prompt`          | string | Yes      | --         | Text description of the video (1-32,000 chars)   |
| `model`           | string | No       | `sora-2`   | Model to use for generation                      |
| `seconds`         | string | No       | `"4"`      | Duration: `"4"`, `"8"`, or `"12"`                |
| `size`            | string | No       | `720x1280` | Resolution as WxH                                |
| `input_reference` | file   | No       | --         | Reference image (JPEG, PNG, WebP) as first frame |

### Allowed `size` Values

| Size        | Orientation | Available Models       |
|-------------|-------------|------------------------|
| `720x1280`  | Portrait    | sora-2, sora-2-pro     |
| `1280x720`  | Landscape   | sora-2, sora-2-pro     |
| `1024x1792` | Portrait    | sora-2-pro only        |
| `1792x1024` | Landscape   | sora-2-pro only        |

### Allowed `seconds` Values

```
"4"   -- 4-second clip
"8"   -- 8-second clip
"12"  -- 12-second clip
```

Note: The `seconds` parameter is passed as a string, not an integer.

### Input Reference Image Requirements

- Must match the target video resolution (size parameter)
- Supported formats: `image/jpeg`, `image/png`, `image/webp`
- The image acts as the first frame of the generated video
- Useful for preserving brand assets, characters, or specific environments
- When using an image, crop it to the chosen aspect ratio before sending

### GET /videos -- List Videos

| Parameter | Type   | Required | Description                            |
|-----------|--------|----------|----------------------------------------|
| `after`   | string | No       | Cursor for pagination                  |
| `limit`   | int    | No       | Number of results per page             |
| `order`   | string | No       | Sort order: `"asc"` or `"desc"`        |

### GET /videos/{video_id}/content -- Download Content

| Parameter | Type   | Required | Default   | Description                            |
|-----------|--------|----------|-----------|----------------------------------------|
| `variant` | string | No       | `"video"` | `"video"`, `"thumbnail"`, `"spritesheet"` |

- `video` -- Returns the MP4 binary stream
- `thumbnail` -- Returns a WebP thumbnail image
- `spritesheet` -- Returns a JPG spritesheet of frames

### POST /videos/{video_id}/remix -- Remix Video

| Parameter | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `prompt`  | string | Yes      | New/modified text description for the remix |

---

## 6. Async Workflow (Create, Poll, Download)

Video generation is asynchronous. A single render can take 30 seconds to 5 minutes depending
on model, resolution, and API load. The workflow has three phases:

### Phase 1: Create the Job

Submit a POST request to `/videos`. You receive a Video object with an `id` and initial
`status` of `"queued"`.

### Phase 2: Poll for Status

Repeatedly call GET `/videos/{video_id}` until `status` transitions to `"completed"` or
`"failed"`. The `progress` field (0-100) indicates approximate completion percentage.

Status lifecycle:
```
queued --> in_progress --> completed
                      \-> failed
```

Valid status values:
- `"queued"` -- Job is waiting to be processed
- `"in_progress"` -- Job is actively rendering
- `"completed"` -- Video is ready for download
- `"failed"` -- Generation failed (check `error` field)

Polling best practices:
- Poll every 5-10 seconds initially
- Use exponential backoff for production systems
- Respect the `openai-poll-after-ms` response header if present
- Set a maximum timeout (recommended: 10 minutes)

### Phase 3: Download the Result

Once status is `"completed"`, call GET `/videos/{video_id}/content` to download the MP4.

Download URLs/assets expire after approximately 1 hour. If you need the video later, download
and store it immediately.

---

## 7. Python SDK Usage

### Installation

```bash
pip install openai>=1.51.0
```

### Basic Text-to-Video

```python
from openai import OpenAI

client = OpenAI()  # Uses OPENAI_API_KEY env var

# Create a video generation job
response = client.videos.create(
    model="sora-2",
    prompt="A cat playing with a ball of yarn in a sunny garden",
    seconds="8",
    size="1280x720"
)

print(f"Job ID: {response.id}")
print(f"Status: {response.status}")
```

### Create and Poll (Convenience Method)

```python
from openai import OpenAI

client = OpenAI()

# This blocks until the video is completed or failed
video = client.videos.create_and_poll(
    model="sora-2",
    prompt="A dramatic sunset over a mountain lake with reflections",
    seconds="8",
    size="1280x720"
)

if video.status == "completed":
    print(f"Video completed! ID: {video.id}")
else:
    print(f"Video failed: {video.error}")
```

### Manual Polling

```python
import time
from openai import OpenAI

client = OpenAI()

# Step 1: Create the job
video = client.videos.create(
    model="sora-2-pro",
    prompt="A timelapse of a flower blooming in a meadow",
    seconds="12",
    size="1792x1024"
)

job_id = video.id
print(f"Job created: {job_id}")

# Step 2: Poll for completion
timeout = 600  # 10 minutes
start_time = time.time()

while (time.time() - start_time) < timeout:
    video = client.videos.retrieve(job_id)
    print(f"Status: {video.status}, Progress: {video.progress}%")

    if video.status == "completed":
        print("Video generation completed!")
        break
    elif video.status == "failed":
        print(f"Video generation failed: {video.error}")
        break

    time.sleep(10)  # Poll every 10 seconds
else:
    print("Timeout waiting for video generation")
```

### Downloading the Video

```python
# Step 3: Download the completed video
if video.status == "completed":
    # Download the MP4
    content = client.videos.download_content(
        video.id,
        variant="video"
    )

    # Write to file
    output_path = "output.mp4"
    with open(output_path, "wb") as f:
        f.write(content.content)
    print(f"Video saved to {output_path}")

    # Optionally download thumbnail
    thumbnail = client.videos.download_content(
        video.id,
        variant="thumbnail"
    )
    with open("thumbnail.webp", "wb") as f:
        f.write(thumbnail.content)
```

### Image-to-Video

```python
from openai import OpenAI

client = OpenAI()

# Open the reference image
with open("reference_image.jpg", "rb") as img_file:
    video = client.videos.create(
        model="sora-2-pro",
        prompt="The scene comes to life with gentle wind blowing through the trees",
        seconds="8",
        size="1280x720",
        input_reference=img_file
    )

print(f"Image-to-video job created: {video.id}")
```

### Remixing an Existing Video

```python
from openai import OpenAI

client = OpenAI()

# Remix an existing video with a new prompt
remixed = client.videos.remix(
    video_id="video_abc123",
    prompt="Change the color palette to warm autumn tones"
)

print(f"Remix job created: {remixed.id}")
```

### Listing and Deleting Videos

```python
from openai import OpenAI

client = OpenAI()

# List recent videos
videos = client.videos.list(limit=10, order="desc")
for v in videos:
    print(f"ID: {v.id}, Status: {v.status}, Created: {v.created_at}")

# Delete a video
client.videos.delete("video_abc123")
```

### Full End-to-End Example

```python
import time
from pathlib import Path
from openai import OpenAI

def generate_video(
    prompt: str,
    model: str = "sora-2",
    seconds: str = "8",
    size: str = "1280x720",
    output_path: str = "output.mp4",
    reference_image_path: str = None,
    timeout: int = 600,
    poll_interval: int = 10
) -> str:
    """
    Generate a video using the OpenAI Sora API.

    Args:
        prompt: Text description of the desired video
        model: Model to use (sora-2 or sora-2-pro)
        seconds: Duration ("4", "8", or "12")
        size: Resolution ("1280x720", "720x1280", "1792x1024", "1024x1792")
        output_path: Where to save the resulting MP4
        reference_image_path: Optional path to a reference image
        timeout: Maximum wait time in seconds
        poll_interval: Seconds between status checks

    Returns:
        Path to the saved video file

    Raises:
        TimeoutError: If generation exceeds timeout
        RuntimeError: If generation fails
    """
    client = OpenAI()

    # Build creation kwargs
    create_kwargs = {
        "model": model,
        "prompt": prompt,
        "seconds": seconds,
        "size": size,
    }

    # Add reference image if provided
    if reference_image_path:
        img_file = open(reference_image_path, "rb")
        create_kwargs["input_reference"] = img_file

    try:
        # Step 1: Create the generation job
        video = client.videos.create(**create_kwargs)
        job_id = video.id
        print(f"[CREATE] Job ID: {job_id}")

        # Step 2: Poll for completion
        start_time = time.time()
        while (time.time() - start_time) < timeout:
            video = client.videos.retrieve(job_id)
            elapsed = int(time.time() - start_time)
            print(
                f"[POLL] {elapsed}s elapsed | "
                f"Status: {video.status} | "
                f"Progress: {video.progress}%"
            )

            if video.status == "completed":
                break
            elif video.status == "failed":
                error_msg = str(video.error) if video.error else "Unknown error"
                raise RuntimeError(f"Video generation failed: {error_msg}")

            time.sleep(poll_interval)
        else:
            raise TimeoutError(
                f"Video generation timed out after {timeout} seconds"
            )

        # Step 3: Download the video
        content = client.videos.download_content(job_id, variant="video")
        Path(output_path).write_bytes(content.content)
        print(f"[DONE] Video saved to: {output_path}")

        return output_path

    finally:
        if reference_image_path and "img_file" in locals():
            img_file.close()


# Usage examples:

# Text-to-video
generate_video(
    prompt="A drone shot flying over a misty forest at sunrise",
    model="sora-2",
    seconds="8",
    size="1280x720",
    output_path="forest_sunrise.mp4"
)

# Image-to-video
generate_video(
    prompt="The scene comes alive with gentle movement and ambient sounds",
    model="sora-2-pro",
    seconds="12",
    size="1792x1024",
    output_path="animated_scene.mp4",
    reference_image_path="my_image.jpg"
)
```

### Async Python Usage

```python
import asyncio
from openai import AsyncOpenAI

async def generate_video_async(prompt: str) -> str:
    client = AsyncOpenAI()

    video = await client.videos.create(
        model="sora-2",
        prompt=prompt,
        seconds="8",
        size="1280x720"
    )

    # Poll until done
    while video.status in ("queued", "in_progress"):
        await asyncio.sleep(10)
        video = await client.videos.retrieve(video.id)
        print(f"Progress: {video.progress}%")

    if video.status == "completed":
        content = await client.videos.download_content(video.id, variant="video")
        output = f"video_{video.id}.mp4"
        with open(output, "wb") as f:
            f.write(content.content)
        return output

    raise RuntimeError(f"Failed: {video.error}")

# Run it
asyncio.run(generate_video_async("A robot dancing in a neon-lit city"))
```

---

## 8. REST API (curl) Usage

### Create Video (Text-to-Video)

```bash
curl -X POST https://api.openai.com/v1/videos \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sora-2",
    "prompt": "A cat playing piano in a jazz bar",
    "seconds": "8",
    "size": "1280x720"
  }'
```

### Create Video (Image-to-Video, multipart)

```bash
curl -X POST https://api.openai.com/v1/videos \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F "model=sora-2-pro" \
  -F "prompt=The scene comes alive with gentle movement" \
  -F "seconds=8" \
  -F "size=1280x720" \
  -F "input_reference=@reference_image.jpg"
```

### Check Status

```bash
curl https://api.openai.com/v1/videos/video_abc123 \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

### Download Completed Video

```bash
curl https://api.openai.com/v1/videos/video_abc123/content?variant=video \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  --output output.mp4
```

### List Videos

```bash
curl "https://api.openai.com/v1/videos?limit=10&order=desc" \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

### Delete Video

```bash
curl -X DELETE https://api.openai.com/v1/videos/video_abc123 \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

### Remix Video

```bash
curl -X POST https://api.openai.com/v1/videos/video_abc123/remix \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Change the lighting to golden hour"
  }'
```

---

## 9. Webhook Notifications

Instead of polling, you can configure webhooks to receive notifications when a video job
finishes.

### Setup

Configure webhooks in the OpenAI platform dashboard at:
https://platform.openai.com/settings/webhooks

You configure:
- A name for the endpoint (for your reference)
- A public URL to a server you control

### Event Types

| Event              | Trigger                            |
|--------------------|------------------------------------|
| `video.completed`  | Video generation finished successfully |
| `video.failed`     | Video generation encountered an error  |

### Webhook Payload Structure

```json
{
  "id": "evt_abc123",
  "object": "event",
  "created_at": 1758941500,
  "type": "video.completed",
  "data": {
    "id": "video_68d7512d..."
  }
}
```

### Verification

Webhook requests include headers that can be used with your webhook secret key to verify
the request originated from OpenAI. Use the `webhook-id` header as an idempotency key to
deduplicate, as OpenAI may occasionally deliver duplicate events.

### Known Limitations (as of early 2026)

There have been reported discrepancies between the documentation and the webhook dashboard.
Some users report that `video.completed` and `video.failed` events were not initially
available in the dashboard settings, though OpenAI has stated this has been resolved.
If webhooks are unreliable, fall back to polling.

---

## 10. Response Objects

### Video Object

```json
{
  "id": "video_68d7512d07848190b3e45da0ecbebcde004da08e1e0678d5",
  "object": "video",
  "status": "in_progress",
  "model": "sora-2-pro",
  "prompt": "A dramatic sunset over a mountain lake",
  "progress": 33,
  "seconds": "8",
  "size": "1280x720",
  "created_at": 1758941485,
  "completed_at": null,
  "expires_at": null,
  "error": null,
  "remixed_from_video_id": null
}
```

### Video Object Fields

| Field                  | Type                | Description                                        |
|------------------------|---------------------|----------------------------------------------------|
| `id`                   | string              | Unique identifier for the video job                |
| `object`               | string              | Always `"video"`                                   |
| `status`               | string              | `"queued"`, `"in_progress"`, `"completed"`, `"failed"` |
| `model`                | string              | Model used for generation                          |
| `prompt`               | string or null      | The prompt used to generate the video              |
| `progress`             | integer             | Approximate completion percentage (0-100)          |
| `seconds`              | string              | Duration of the generated clip                     |
| `size`                 | string              | Resolution of the generated video                  |
| `created_at`           | integer             | Unix timestamp when the job was created            |
| `completed_at`         | integer or null     | Unix timestamp when the job finished               |
| `expires_at`           | integer or null     | Unix timestamp when downloadable assets expire     |
| `error`                | object or null      | Error details if generation failed                 |
| `remixed_from_video_id`| string or null      | Source video ID if this is a remix                 |

### Error Object

```json
{
  "code": "content_policy_violation",
  "message": "The prompt was flagged by content moderation"
}
```

### Video Job Lifecycle (Azure variant, for reference)

The Azure OpenAI implementation shows additional intermediate states:
```
queued --> preprocessing --> running --> processing --> succeeded
                                                  \-> failed
                                                  \-> cancelled
```

The direct OpenAI API uses the simpler:
```
queued --> in_progress --> completed / failed
```

---

## 11. Pricing

### API Pay-Per-Second

| Model        | Resolution         | Cost per Second |
|--------------|--------------------|-----------------|
| sora-2       | 1280x720 / 720x1280| $0.10           |
| sora-2-pro   | 1280x720 / 720x1280| $0.30           |
| sora-2-pro   | 1792x1024 / 1024x1792 | $0.50        |

### Cost Examples

| Scenario                  | Model      | Duration | Resolution | Cost   |
|---------------------------|------------|----------|------------|--------|
| Quick social media clip   | sora-2     | 4s       | 720p       | $0.40  |
| Standard social clip      | sora-2     | 8s       | 720p       | $0.80  |
| Extended standard clip    | sora-2     | 12s      | 720p       | $1.20  |
| Pro social clip           | sora-2-pro | 8s       | 720p       | $2.40  |
| Pro HD production clip    | sora-2-pro | 12s      | 1024p      | $6.00  |

### Important Pricing Notes

- You are charged for EVERY generation attempt, including those you do not use
- Failed generations may still incur charges
- Iteration/experimentation costs add up quickly
- Budget for 3-5x the cost of your final video to account for iterations

---

## 12. Rate Limits

| Access Level   | Requests/Minute | Requests/Day | Notes            |
|----------------|-----------------|--------------|------------------|
| API Tier 2     | 10 RPM          | 200/day      | Standard queue   |
| API Tier 3     | 30 RPM          | 600/day      | Priority queue   |
| API Tier 4     | 60 RPM          | 1,200/day    | Priority queue   |
| Enterprise     | 200+ RPM        | Custom       | Dedicated        |
| ChatGPT Plus   | 5 RPM           | 50/day       | Via ChatGPT UI   |
| ChatGPT Pro    | 50 RPM          | 500/day      | Priority queue   |

API tiers auto-upgrade based on consistent usage and verified payment history.

---

## 13. Content Restrictions

The following content restrictions apply to all Sora video generation:

- Content must be age-appropriate (suitable for under-18 audiences)
- No copyrighted characters or music in prompts
- No real people or public figures
- Face-containing reference images may be rejected
- Prompts are subject to OpenAI's content moderation system
- Requests with harmful content are rejected with a `content_policy_violation` error

---

## 14. Error Handling

### HTTP Error Codes

| Code | Error                     | Action                                    |
|------|---------------------------|-------------------------------------------|
| 400  | Bad Request               | Check parameters; may be content policy   |
| 401  | Unauthorized              | Verify API key                            |
| 403  | Forbidden                 | Check account access/permissions          |
| 404  | Not Found                 | Invalid video_id                          |
| 429  | Too Many Requests         | Rate limited; implement backoff           |
| 500  | Internal Server Error     | Retry with exponential backoff            |

### Retry Strategy

- DO retry on 429 (rate limit) and 5xx (server errors) with exponential backoff
- DO NOT retry on 400 (bad request) or 401 (auth) errors -- these require fixing the request
- Content policy violations (400) require modifying the prompt

### Recommended Exponential Backoff

```python
import time
import random

def retry_with_backoff(func, max_retries=5, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            print(f"Attempt {attempt + 1} failed, retrying in {delay:.1f}s: {e}")
            time.sleep(delay)
```

---

## 15. Azure OpenAI Variant

Azure OpenAI has its own implementation of the Sora API with different endpoints and
parameter names.

### Azure Endpoints

```
POST   {endpoint}/openai/v1/video/generations/jobs?api-version=preview
GET    {endpoint}/openai/v1/video/generations/jobs/{job_id}?api-version=preview
GET    {endpoint}/openai/v1/video/generations/{generation_id}/content/video?api-version=preview
```

### Azure Parameter Differences

| OpenAI Direct | Azure OpenAI    | Notes                     |
|---------------|-----------------|---------------------------|
| `seconds`     | `n_seconds`     | Duration (5-20 seconds)   |
| `size`        | `width` + `height` | Separate dimension fields |
| --            | `n_variants`    | 1-4 variants per request  |
| --            | `inpaint_items` | Image/video input config  |

### Azure Supported Resolutions

480x480, 720x720, 1080x1080, 1280x720, 1920x1080

### Azure Authentication

```python
# API Key
headers = {"api-key": api_key, "Content-Type": "application/json"}

# Microsoft Entra ID (keyless)
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()
token = credential.get_token("https://cognitiveservices.azure.com/.default")
headers = {"Authorization": f"Bearer {token.token}", "Content-Type": "application/json"}
```

### Azure Status Values

```
queued --> preprocessing --> running --> processing --> succeeded / failed / cancelled
```

---

## Appendix A: Official Resources

- OpenAI Video Generation Guide: https://platform.openai.com/docs/guides/video-generation
- OpenAI API Reference (Videos): https://platform.openai.com/docs/api-reference/videos
- OpenAI Sora 2 Model Page: https://platform.openai.com/docs/models/sora-2
- OpenAI Sora 2 Pro Model Page: https://platform.openai.com/docs/models/sora-2-pro
- OpenAI Official Sample App: https://github.com/openai/openai-sora-sample-app
- OpenAI Python SDK: https://github.com/openai/openai-python
- OpenAI Pricing: https://openai.com/api/pricing/
- Azure OpenAI Sora Quickstart: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/video-generation-quickstart

## Appendix B: Python SDK Method Summary

| Method                        | Description                        | Returns                 |
|-------------------------------|------------------------------------|-------------------------|
| `client.videos.create()`      | Start a new generation job         | `Video`                 |
| `client.videos.create_and_poll()` | Create and wait for completion | `Video`                 |
| `client.videos.poll()`        | Poll an existing job               | `Video`                 |
| `client.videos.retrieve()`    | Get job status/metadata            | `Video`                 |
| `client.videos.list()`        | List all videos (paginated)        | `Page[Video]`           |
| `client.videos.delete()`      | Delete a video                     | `VideoDeleteResponse`   |
| `client.videos.download_content()` | Download video/thumbnail/spritesheet | `BinaryResponse` |
| `client.videos.remix()`       | Create variation of existing video | `Video`                 |

Async variants are available as `client.videos.acreate()`, etc., or via `AsyncOpenAI`.

## Appendix C: Quick Decision Matrix

| Use Case                | Model      | Size       | Seconds | Est. Cost |
|-------------------------|------------|------------|---------|-----------|
| TikTok/Reels vertical   | sora-2     | 720x1280   | "8"     | $0.80     |
| YouTube landscape        | sora-2     | 1280x720   | "12"    | $1.20     |
| Product demo (quality)   | sora-2-pro | 1280x720   | "8"     | $2.40     |
| Cinematic HD             | sora-2-pro | 1792x1024  | "12"    | $6.00     |
| Quick prototype          | sora-2     | 720x1280   | "4"     | $0.40     |
