---
name: jianying-voiceover-cut
description: Batch video editing workflow for Jianying/剪映 smart spoken-word drafts. Use when the user has vertical product口播 videos where an offscreen prompt/画外音 alternates with the on-camera speaker, and wants Codex to read 剪映 “智能剪口播/剪口播” draft timings, remove offscreen voice, keep the on-camera final口播, and export 1080P high-bitrate MP4 files.
---

# Jianying Voiceover Cut

## Purpose

Remove offscreen prompt voice from spoken-word product videos by using Jianying draft timing data instead of guessing from the raw video alone. This skill assumes the user has already put each source clip into a separate Jianying draft and run “智能剪口播/剪口播” so `attachment_script_video.json` contains sentence timings.

This skill does **not** rely on Jianying export. It reads Jianying draft JSON, chooses keep intervals, then renders from the original camera files with `ffmpeg`.

## Standard Workflow

1. Confirm the source folder contains the raw videos and `_剪辑输出` is the desired output folder.
2. Confirm each video has one Jianying draft with “智能剪口播/剪口播” recognition already run and saved.
3. Generate review plans:

```powershell
python "C:\Users\Administrator\.codex\skills\jianying-voiceover-cut\scripts\jianying_voiceover_batch.py" plan --root "<source-folder>"
```

4. Review the generated TSV files in `_codex_work\jianying_voiceover_cut\plans`.
   - `KEEP` usually means on-camera speaker A.
   - `DROP` usually means offscreen prompt B, silence, director talk, or a false start.
   - If needed, edit the matching `*_plan.json`: change row `"keep": true/false`, or add exact `"manual_intervals"`.
5. Render:

```powershell
python "C:\Users\Administrator\.codex\skills\jianying-voiceover-cut\scripts\jianying_voiceover_batch.py" render --root "<source-folder>"
```

For fast routine jobs where the pattern is clear, use `all` to plan and render in one pass:

```powershell
python "C:\Users\Administrator\.codex\skills\jianying-voiceover-cut\scripts\jianying_voiceover_batch.py" all --root "<source-folder>"
```

## Matching Rules

- Match videos to Jianying drafts through `script_video.translate_segments[].material_path`.
- Prefer one video per Jianying draft. If multiple drafts reference the same filename, use the most recently modified script.
- If no script is matched, do not blind-edit the clip; tell the user to run Jianying recognition or locate the missing draft.

## Keep/Drop Heuristics

The script estimates sentence loudness from the original audio and uses Jianying sentence timings.

- High loudness cluster: usually the on-camera speaker A. Keep the final, coherent口播.
- Low loudness cluster: usually the offscreen prompt B. Drop it.
- Drop pure silence markers (`[...1.2s]`), offscreen coaching, and UI/director phrases such as `OK`, `对`, `再来一遍`, `是吧`, `不行是吧`, `刚刚指错`, `可以拿了`.
- When two adjacent sentences are near duplicates, keep the later/louder/final version.
- When Jianying embeds a long pause or repeated false start inside one sentence, use `manual_intervals` in the plan JSON to keep only the clean part.

Always prefer preserving natural pauses inside A’s final delivery over chopping every tiny gap.

## Output Standard

Render from the original file, not from Jianying previews:

- MP4, H.264, AAC
- Vertical 1080P: `1080x1920`
- Video bitrate target around `12000k`, max `16000k`
- Audio `192k`
- Filename suffix: `_删画外音_剪映时间码版_1080P高码率.mp4`

Validate every finished file with `ffprobe` for width, height, duration, and bitrate.

## GUI Automation Note

Do not promise reliable automatic clicking inside Jianying. Jianying’s “智能剪口播” is a desktop UI feature, not a stable command-line API. The production-safe workflow is: user runs recognition in Jianying, Codex reads draft JSON and renders the edits.
