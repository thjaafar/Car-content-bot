# Car Content Bot

Generates faceless AI car-content videos (script → voiceover → visuals → captions)
and lets you review + post them to TikTok from a phone-friendly dashboard.

## What's here

```
app/
  main.py            FastAPI backend + API routes
  storage.py          simple JSON-file job store (no DB needed to start)
  pipeline/
    script_gen.py     writes the voiceover script (OpenAI, swappable for Claude)
    voice_gen.py       text-to-speech (ElevenLabs)
    visuals.py         stock footage (Pexels) + AI clip stub (Runway/Luma)
    assemble.py        ffmpeg: stitches clips, adds voiceover, burns captions
    tiktok.py          TikTok Content Posting API (Direct Post)
static/index.html     the dashboard (plain HTML/JS, no build step)
storage/               generated videos/audio/clips land here
```

## 1. Local setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

You also need **ffmpeg** installed on whatever machine runs this:
- Mac: `brew install ffmpeg`
- Ubuntu/Debian: `sudo apt install ffmpeg`
- Most hosting platforms (Railway, Render) can install it via a buildpack/nixpacks config — ask if you want that file.

Copy `.env.example` to `.env` and fill in keys as you get them. **The app runs
without any keys** — it'll use placeholder script text, a silent audio track,
and a plain background, so you can test the whole flow (create job → see it
move through statuses → get a video file) before wiring up real services.

Run it:
```bash
uvicorn app.main:app --reload --port 8000
```
Open `http://localhost:8000` — that's your dashboard.

## 2. Getting your API keys

| Service | Free tier? | Used for |
|---|---|---|
| [platform.openai.com](https://platform.openai.com) | Pay-as-you-go, cheap | script writing |
| [elevenlabs.io](https://elevenlabs.io) | Yes, limited | voiceover |
| [pexels.com/api](https://pexels.com/api) | Yes, free | stock car footage |
| Runway or Luma | Paid | AI-generated clips (optional — the app works with stock-only) |

Add each key to `.env` as you sign up; the app picks them up automatically,
no code changes needed.

## 3. Connecting TikTok (the part that takes longest)

TikTok doesn't let a brand-new app auto-post publicly right away. Here's the
real path:

1. Create an app at [developers.tiktok.com](https://developers.tiktok.com) and add the **Content Posting API** product.
2. Get `TIKTOK_CLIENT_KEY` / `TIKTOK_CLIENT_SECRET` from that app, add to `.env`.
3. Do TikTok's OAuth login flow **once** for your own account to get a
   long-lived `TIKTOK_ACCESS_TOKEN` (TikTok's docs have a copy-pasteable
   example for this — happy to build this login screen into the dashboard
   next if you want it in-app instead of doing it manually).
4. Until TikTok **audits and approves** your app (their review, typically a
   couple weeks, evaluating your actual use case), posts can only go out as
   **private/self-only** — that's why `tiktok.py` defaults to
   `privacy_level: SELF_ONLY`. You can review them in the TikTok app and
   manually flip to public, or wait for approval to post public directly.
5. Apply for the audit once you're ready to go fully public/automated — TikTok
   wants to see a working demo, so having this dashboard running helps.

**In the meantime**, the dashboard still does everything except the final
public post: generate, review, download, and manually upload if you'd rather
not wait on the audit.

## 4. Deploying so you can check it from your phone

Simplest path: [Railway](https://railway.app) or [Render](https://render.com).
Both support:
- Point at this repo
- Set the same env vars from `.env` in their dashboard
- Add a `nixpacks.toml` or Docker step to install ffmpeg (ask me to generate one for whichever platform you pick)
- They give you a public URL — that's your dashboard, bookmark it on your phone

## 5. Using it

1. Open the dashboard, type a topic ("why turbos whistle", "cheapest way to
   mod a Civic"), hit **Generate**.
2. Watch it move through `scripting → voicing → visuals → assembling → ready`
   (polls every 4s automatically).
3. Preview the video right in the dashboard.
4. Hit **Post to TikTok** (posts as private/self-only until you're audited),
   or download and post manually.

## Notes on account health

Fully automated, fully AI, zero-touch posting is exactly the pattern TikTok's
spam detection watches for. Leaving a manual review tap in the loop (which
this dashboard does by design), varying posting times, and occasionally mixing
in non-AI content all meaningfully help an account survive and grow.
