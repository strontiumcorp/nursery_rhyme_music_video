# Nursery‑Rhyme Music Video Automation — CloneVoice → Artistly → VideoExpress

An autonomous, browser‑based agent workflow that turns **one idea + one ratio** into a finished nursery‑rhyme music video and submits it to VideoExpress's render queue — end to end, no manual clicking.

**Pipeline:** generate the song in **CloneVoice.ai** → build a character‑consistent storyboard in **Artistly.ai** → generate one **VideoExpress.ai** clip per storyboard scene → assemble all clips on the timeline → add the music → **match the video endpoint exactly to the audio endpoint** → save → export FullHD MP4.

---

## 🎬 Example output (this exact workflow, verified run)

**Exported video:** https://cdn-ny-b.videoexpress.ai/video/1786603107_6a7d6663c966b.mp4

| | |
|---|---|
| Song | *Jannat and Her Toys* (CloneVoice, Kids‑Rhyme, English, 139.22 s) |
| Storyboard | 35 scenes, Artistly **Music Storyboard** agent, 16:9 |
| Video | 35 clips (all normal 4.04 s — no booster needed), 3D animation |
| Sync | video endpoint == audio endpoint, **0‑pixel difference** (final clip exact‑trimmed by 2.23 s) |
| Export | **Landscape 16:9 · High · FullHD · mp4** |

---

## ✍️ The prompt used

The agent asks exactly two questions, then runs autonomously. In this run the inputs were:

> **1. Idea/prompt:** `Jannat is a 2 years old girl. She has lots of toys. He is enjoying by playing.`
>
> **2. Ratio:** `Landscape`

*(Note the stray "He" in that idea — the identity lock still held Jannat as a girl across all 35 scenes, because identity is resolved once from the idea and re‑asserted in the storyboard prompt rather than re‑read per scene.)*

Everything else — song title, lyrics, music style/language, protagonist identity lock, export name, story arc, and all visual treatment — was **inferred** from that one idea. You can add as much or as little detail to the idea as you like (protagonist gender/age/appearance/clothing, setting, action, mood, music style, language); anything omitted is inferred. **Only the ratio is never guessed.**

---

## 🚀 How to use

1. Make sure **CloneVoice.ai, Artistly.ai, and VideoExpress.ai are already signed in** in the browser the agent controls. (The workflow never handles credentials or API keys.)
2. Start the agent with the system prompt in [`SYSTEM_PROMPT.md`](SYSTEM_PROMPT.md) (or trigger phrase **"Generate nursery rhymes music video"**).
3. Answer the **two questions**: your idea, and **Landscape (16:9)** or **Vertical (9:16)**.
4. The agent presents a short production brief and then runs to completion on its own — it never asks you to say "ready"/"continue".
5. It finishes when VideoExpress shows **"Your movie creation is currently number N in the queue."** The MP4 appears under **My Videos** once background rendering completes.

Typical wall‑clock: a few minutes for the song, a few for the storyboard, then batched video generation (5 concurrent) — most of the time is the render queue, not interaction.

---

## 📁 Files in this folder

| File | Purpose |
|---|---|
| [`SYSTEM_PROMPT.md`](SYSTEM_PROMPT.md) | The agent's operating instructions. **§0** is the mandatory device‑agnostic interaction contract; **§17** is the exact DOM/API selector reference; **§18–19** cover validation checkpoints, support investigation, and the golden invariants. |
| [`WORKFLOW.json`](WORKFLOW.json) | Machine‑readable config (v2.0.0): inputs, 38 ordered steps, `execution_mechanics`, `dom_api_contract`, `validation_checkpoints`, `support_investigation`, gates, and safety rules. |
| [`RUNTIME_STATE.example.json`](RUNTIME_STATE.example.json) | A **real completed‑run** checkpoint file — a worked example of the resumable, human‑readable state the agent maintains. |
| `README.md` | This file. |

---

## 🔒 Key requirements & guarantees

- **Ratio is a project‑wide invariant** — every image, clip, canvas, and the export use the one chosen orientation; landscape and vertical are never mixed.
- **Identity lock** — the protagonist's gender, age, face, hair, core clothing, and visual medium stay consistent across all scenes; a wrong‑character storyboard is rejected and regenerated, never animated.
- **Storyboard agent priority & fallback** — Artistly **Nursery Rhymes** always runs first (high priority). It sometimes errors or returns a **single image instead of a full storyboard**, so every attempt is validated (multi-scene, matches the music theme, identity intact) with up to **3 attempts**; each failure is logged to `RUNTIME_STATE.json` `error_history` for later debugging, and only after the third failure does the run fall back to **Music Storyboard** (low priority) with the character-lock prompt — validated the same way, also with up to 3 attempts.
- **Closed-mouth (no-speech) pipeline — STRICT (v2.4.0)** — mouth control starts at the stills: a storyboard where more than ~10% of frames show an open mouth is regenerated (verified root cause: Image-To-Video continues the source frame, so an open-mouth still becomes a talking clip no matter what the prompt says). Video prompts are sanitized of vocal trigger words ("singing", "laughing"…) and wrapped in closed-mouth prefix/suffix language — every prompt ends with the literal phrase **"No Mouth Movement or lip-sync"** — prompt enhancement and lip sync stay off, the style dropdown (which defaults to Human) is always set to 3D, and **every** clip is watched for mouth motion — with a recorded verdict — before it reaches the timeline.
- **Exact sync** — the final video endpoint equals the audio endpoint with **zero‑pixel tolerance** (achieved by the playhead‑set + cut + tail‑delete method).
- **`N` is dynamic** — however many storyboard scenes are produced, exactly that many timeline clips are made (no fixed count).
- **≤ 5 concurrent generations**, batched, with a completion barrier between batches.
- **Device‑agnostic** — actions use DOM selectors + JavaScript event dispatch (never screenshot pixel coordinates), so it works across screen sizes, zoom levels, and browsers.

---

## 🛠️ Resumability & support

The agent writes a durable **`RUNTIME_STATE.json`** (see the `.example` file) after every verified side effect, with a per‑gate checkpoint `{gate, status, evidence, method, timestamp, artifact_ids}`.

If a run fails and a user reports it, follow **`support_investigation`** in `WORKFLOW.json`:

1. Load `RUNTIME_STATE.json`.
2. Find the first gate whose `status` isn't `pass`.
3. Read its `evidence` + surrounding `error_history`.
4. Re‑verify that gate live via the matching read API in `dom_api_contract` (auth, folder contents, job status, endpoints, queue) — app state is authoritative.
5. Resume from that gate's `next_safe_action` idempotently — never repeat a verified side effect.

Gates cover: **auth · CloneVoice · identity/image‑consistency · storyboard · import · batch submission · batch completion · timeline · audio · sync · ratio · save · export.**

Known non‑blockers (do **not** stop): visible spinners/percentages, a stale success toast, or an MCP/API `419` (use the browser DOM instead). Real blockers: authentication, CAPTCHA, credits/payment, unavailable account access, or an explicit unrecoverable error.
