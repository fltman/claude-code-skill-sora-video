# Sora Video Generation Skill for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that generates AI videos using OpenAI's Sora API.

## Features

- **Text-to-video** — Create video clips from text descriptions
- **Image-to-video** — Animate a reference image into a video clip
- Models: `sora-2` (fast) and `sora-2-pro` (production quality)
- Durations: 4, 8, or 12 seconds
- Resolutions: 720p and 1024p HD

## Setup

1. Install the dependency:
   ```bash
   pip install openai
   ```

2. Set your OpenAI API key:
   ```bash
   export OPENAI_API_KEY="your-key"
   ```

3. Add this skill to your Claude Code project in `.claude/settings.local.json`:
   ```json
   {
     "permissions": {
       "allow": [
         "Bash(python*skills/sora-video/scripts/generate_video.py*)",
         "Bash(pip install openai*)"
       ]
     }
   }
   ```

## Usage

### Text-to-video

```bash
python skills/sora-video/scripts/generate_video.py \
  --prompt "A drone shot flying over a misty forest at sunrise" \
  --output forest.mp4
```

### Image-to-video

```bash
python skills/sora-video/scripts/generate_video.py \
  --input photo.jpg \
  --prompt "The scene comes to life with gentle wind" \
  --output animated.mp4
```

### Pro model, HD

```bash
python skills/sora-video/scripts/generate_video.py \
  --prompt "A cinematic sunset over mountains" \
  --model sora-2-pro \
  --size 1792x1024 \
  --seconds 12 \
  --output cinematic.mp4
```

## Arguments

| Argument | Short | Default | Description |
|----------|-------|---------|-------------|
| `--prompt` | `-p` | — | Text description of the video (required) |
| `--output` | `-o` | `generated_video.mp4` | Output file path |
| `--input` | `-i` | — | Reference image for image-to-video |
| `--model` | `-m` | `sora-2` | `sora-2` or `sora-2-pro` |
| `--seconds` | `-s` | `8` | Duration: `4`, `8`, or `12` |
| `--size` | | `1280x720` | Resolution (see below) |
| `--timeout` | | `600` | Max wait time in seconds |

### Available sizes

| Size | Orientation | Models |
|------|-------------|--------|
| `1280x720` | Landscape | sora-2, sora-2-pro |
| `720x1280` | Portrait | sora-2, sora-2-pro |
| `1792x1024` | Landscape HD | sora-2-pro only |
| `1024x1792` | Portrait HD | sora-2-pro only |

## Pricing

| Model | Resolution | Per Second | 4s | 8s | 12s |
|-------|-----------|------------|-----|-----|------|
| sora-2 | 720p | $0.10 | $0.40 | $0.80 | $1.20 |
| sora-2-pro | 720p | $0.30 | $1.20 | $2.40 | $3.60 |
| sora-2-pro | 1024p | $0.50 | $2.00 | $4.00 | $6.00 |

## License

MIT
