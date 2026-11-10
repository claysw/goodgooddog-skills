# commentary-to-figjam

Record your app while you talk through it. Point at things with the cursor and say what you see: "this is confusing", "I'd like suggestions here", "keep this". Hand the recording to Claude Code and it builds a FigJam board with one screenshot and one note per key moment.

Each note says what you said, what Claude thinks it means, and where in the app it is. Notes are coloured by type: Bug, Friction, Idea, Question or Keep. Moments sit in a grid three wide, in the order you talked about them.

The board is the review step. Edit or delete notes there, explain anything Claude marked low confidence, then ask for tickets. Claude groups them under one parent ticket, links each ticket to its note and each note back to its ticket.

## What you need

- **Claude Code on your own computer.** Cloud sandboxes usually block the screenshot upload to Figma.
- **The Figma MCP connector**, connected to an account that can create FigJam files.
- **ffmpeg** and **uv**. On macOS: `brew install ffmpeg uv`.
- **Apple Silicon** for the fastest transcription (Parakeet on MLX). Other machines fall back to faster-whisper, which is slower but works.
- **For tickets:** a tracker connector, such as Linear. The skill has extra tips for Linear but works with any tracker that has an MCP connector.

## Install

```
/plugin marketplace add claysw/goodgooddog-skills
/plugin install commentary-to-figjam@goodgooddog
```

Then give Claude a recording:

```
/commentary-to-figjam:commentary-to-figjam ~/Desktop/walkthrough.mp4
```

With no path, it uses the newest `.mp4` or `.mov` on your Desktop.

## Recording tips

- Talk while you point. "This" and "here" are how Claude finds the right frame, so have the cursor on the thing as you say it.
- One thought per sentence or two. Claude merges rambles about one thing into one note.
- A few minutes is plenty. Expect about one note per 30 to 60 seconds of commentary.

## Settings

The first time, Claude asks for anything it can't work out (which Figma team, which tracker and team, how its labels map to note types) and offers to save the answers in your project's `CLAUDE.md`:

```markdown
## commentary-to-figjam settings
- App: Acme dashboard
- Figma plan: team::1234567890 (Acme)
- Tracker: Linear, team ACME
- Labels: Bug → Bug; Friction, Idea → Feature (Improvement for a tweak); Question → needs-info; triage → needs-triage
- Filing docs: docs/issue-tracker.md
```

Working files (audio, transcript, frames, `moments.json`) go in `~/Documents/commentary-to-figjam/<date>-<topic>/` unless the settings name another folder.
