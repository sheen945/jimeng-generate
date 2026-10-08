# jimeng-generate

The single, default AI image/video generation channel for this machine: it calls the Jimeng dual-account service through the local New API gateway. Every request to draw, illustrate, design posters or covers, or generate video must go through this skill — the built-in WorkBuddy ImageGen is explicitly off-limits.

## Introduction

This is a WorkBuddy / Claude Code skill that hard-codes "generate images and videos with Jimeng" into one canonical local pipeline. When the agent receives any image- or video-generation request, it no longer asks for keys or endpoints — it simply follows the link configuration and procedures written into the skill, guaranteeing consistent output quality and artifact placement.

The problem it solves: the free Jimeng API service (jimeng-free-api) runs locally on port 8000 with dual-account rotation and failure cooldowns, but its authentication scheme (`jm_` keys / sessionid) is incompatible with WorkBuddy's unified token system. The New API gateway (port 3000) bridges that gap — but video generation must go through `chat/completions` rather than `videos/generations`, dual-account cooldowns occasionally surface as 1015 errors, and responses contain expiring signed URLs. Every one of these pitfalls has a documented fix inside the skill.

Who it's for: users who run jimeng-free-api plus a New API gateway locally and want their agent to call Jimeng reliably for image and video generation.

## Features

- **Single-channel mandate**: all "draw an image / generate a picture / make a graphic / poster / cover / text-to-image" and video requests go through this skill first; the built-in WorkBuddy ImageGen is banned. Trigger words include: 即梦画图, 即梦生成, 画个, 画一张, 生成一张图, 帮我做图, jimeng 生图/生视频.
- **Three-tier link architecture**: WorkBuddy → New API gateway (`http://127.0.0.1:3000/v1`, `sk-` token auth) → Jimeng service (`http://localhost:8000`, auto-started at boot via the JimengFreeAPI-Autostart scheduled task, which internally handles dual-account rotation and failure cooldown). Never hit port 8000 directly with an `sk-` key — it returns a 1015 login-verification error.
- **Image generation**: 9 image models (jimeng-image-3.0 through 5.0-pro). The rule is "always use the strongest current model" (field-tested best as of 2026-09: `jimeng-image-5.0-pro`); when in doubt, query `/v1/models` and pick the highest-versioned pro model. Uses the standard `POST /v1/images/generations` endpoint with square/portrait/landscape size options.
- **Video generation**: 11 video models (default `jimeng-video-seedance-2.5`; use `seedance-2.0-fast` when speed matters). Through the 3000 gateway you must use `POST /v1/chat/completions` — the gateway doesn't recognize `/v1/videos/generations` (404). Aspect ratio and duration are controlled via the prompt; with a first-frame reference image, the output follows the reference's aspect ratio.
- **The 1015 retry rule**: when account rotation lands on a cooling-down account, the service returns a `{"code":-2001,...1015}` error wrapped in HTTP 200. You must retry in a loop until the response contains `choices`, waiting 10 seconds between attempts — field tests show success within 4 tries; scripts cap it at 8.
- **Immediate download of expiring links**: both the image response's `data[0].url` and the video response's `![video](...)` URL are **temporary signed links that expire** — download them locally right away. All artifacts go to one fixed directory with the naming convention `topic-YYYYMMDD-HHMM.png`.
- **Image-to-image / reference images**: the reference-image parameter is `filePath` (not `image`), accepting local paths, http(s) URLs, or base64 data URIs — up to 10 images. Prompts that must preserve a person have to spell out gender + age + key features, or the model may swap the person, even their gender.
- **Specialized know-how for IP-character sticker animations** (overlaid onto live-action footage, keyed out in CapCut/Jianying): if the character itself is green, switch from green screen to a magenta `#FF00FF` background; for three-view character reference sheets, route A (feed the whole sheet as the reference — most faithful look, at the cost of trimming the first 0.1–0.2 s and inheriting the sheet's aspect ratio) is recommended over route B (extracting a single front view onto a magenta background as the first frame — field-tested failure: auto-cropping clipped the character's head); prompts must include "camera locked, character fully in frame at center" to prevent push-ins that move the character out of frame.
- **Complete troubleshooting guide**: both causes of 1015 errors (wrong entry point / cooling-down account) and their fixes; port 3000 unresponsive = New API isn't running; port 8000 unresponsive = the Jimeng service is down; account-abnormal / insufficient-quota messages = check the account pool in the 8000 admin console.

## How It Works / Tech Stack

```
WorkBuddy agent
   │  Authorization: Bearer sk-... (New API token)
   ▼
New API gateway   http://127.0.0.1:3000/v1
   │  (adapter: OpenAI-compatible API → Jimeng protocol)
   ▼
jimeng-free-api   http://localhost:8000
   │  (dual-account rotation, automatic failure cooldown;
   │   auto-started at boot via scheduled task)
   ▼
Jimeng image / video generation service
```

- Images: `POST /v1/images/generations` (standard OpenAI image-API shape), ~10–20 seconds per image, 180-second timeout.
- Videos: `POST /v1/chat/completions` (multimodal message body — text only for pure text-to-video, add an `image_url` entry in the content array for a reference frame), roughly 1.5–5 minutes per 5-second clip.
- Invocation: curl or any HTTP client; every command in the skill is copy-paste runnable.

## Installation & Usage

Copy this repository's directory into your skills folder, keeping the folder name `jimeng-generate`:

- WorkBuddy / CodeBuddy: `~/.workbuddy/skills/jimeng-generate/`
- Claude Code: `~/.claude/skills/jimeng-generate/`

Restart the session and the trigger words will match automatically.

**Prerequisites**: see `.env.example` — you need a local New API gateway token (environment variable `NEWAPI_TOKEN`), and both services (New API on 3000, jimeng-free-api on 8000) must be running.

**Image example**:

```bash
curl -s -m 180 http://127.0.0.1:3000/v1/images/generations \
  -H "Authorization: Bearer sk-YOUR_NEWAPI_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"model":"jimeng-image-4.0","prompt":"<prompt>","size":"1024x1024","response_format":"url"}'
```

Download `data[0].url` from the response immediately (the link expires).

**Video essentials**: use `POST /v1/chat/completions`; state duration and aspect ratio in the prompt (e.g. "5 seconds, square 1:1 frame"); extract the video URL from `choices[0].message.content` with the regex `https?://[^\s)\]"'\\]+` and download it immediately; on 1015 errors, retry at 10-second intervals.

**Delivery**: after generation, present the local files to the user with `present_files`, noting which model was used, the prompt, and the file path.

## Project Structure

```
jimeng-generate/
├── SKILL.md        # Skill definition (frontmatter + full link config, procedures, troubleshooting)
├── README.md       # Chinese documentation
├── README_EN.md    # This file (English documentation)
├── .env.example    # Environment variable sample (NEWAPI_TOKEN)
└── .gitignore      # Standard exclusion rules
```

Reference scripts mentioned in the skill (e.g. `gen_songbao.py`, `make_first_frame.py`) and the full API documentation (`jimeng-free-api-all/API.md`) live in other local working directories and are not part of this repository.

## Notes & Caveats

- **Strongly bound to this machine's environment**: the link configuration (ports 3000/8000, the artifact output path) reflects local field-tested values. When deploying on a different machine, set up New API + jimeng-free-api first and adjust paths accordingly.
- **Never call port 8000 directly**: the Jimeng service only accepts its own `jm_` keys or plaintext sessionids; an `sk-` token sent directly always fails with 1015. All calls must go through the 3000 gateway.
- **Link expiry**: image and video responses contain temporary signed URLs — download them immediately or lose them.
- **Model freshness**: "the strongest current model" should be determined by a live `/v1/models` query — don't hard-code older model versions.
- **IP-asset discipline**: unless the user explicitly asks, do not pre-process their IP reference images (cropping / background swaps) — route B has a documented failure.
- Tokens are sensitive: never commit a real `sk-` token to the repository (`.env` is already excluded via `.gitignore`).

## License

MIT

## Author

sheen945
