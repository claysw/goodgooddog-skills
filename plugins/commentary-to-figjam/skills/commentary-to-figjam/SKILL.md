---
name: commentary-to-figjam
description: Turn a screen recording with spoken commentary on an app into a new FigJam board, with one screenshot and a ticket-ready note per key moment, then optionally file the notes as tickets. Use when the user shares a screen recording (.mp4 or .mov) of an app they talked over, or asks to turn a walkthrough, demo or feedback recording into a board, notes or tickets.
---

# Commentary to FigJam

The user records an app while talking through what they see (UI behavior, bugs, ideas) and points at things with the cursor. This skill transcribes the audio, finds the key moments, grabs a frame for each, and builds a **new FigJam board** where each moment is a screenshot plus a note that can later become a ticket.

**Run it locally** (Claude Code on the user's own computer). Cloud containers usually block the screenshot upload (see "Running in a cloud container" at the end).

Do not file tickets in this skill's main flow. The board is the review step. Filing happens only when the user asks (see "Later: turning notes into tickets").

## Settings

Look for a `commentary-to-figjam` settings block in the project's `CLAUDE.md` (or the user's global one). Use whatever it sets and the defaults for the rest:

| Setting | Default |
|---|---|
| App name, used in the board title | Ask, or infer from the screens |
| Figma plan key | From `whoami`: the only plan, or ask which |
| Working folder | `~/Documents/commentary-to-figjam/` |
| Issue tracker and team | Ask when the user first asks for tickets |
| Labels for Bug, Friction, Idea, Question, and a triage label | Ask, after listing the team's labels |
| Project docs on filing issues | None |

When you had to ask, offer once to save the answers as a settings block in `CLAUDE.md`, for example:

```markdown
## commentary-to-figjam settings
- App: Acme dashboard
- Figma plan: team::1234567890 (Acme)
- Tracker: Linear, team ACME
- Labels: Bug → Bug; Friction, Idea → Feature (Improvement for a tweak); Question → needs-info; triage → needs-triage
- Filing docs: docs/issue-tracker.md
```

## 0. Prerequisites (check once per run)

- `ffmpeg` and `ffprobe` on PATH. If missing, install with the system's package manager (`brew install ffmpeg` on macOS).
- `uv` on PATH (`brew install uv`, or see docs.astral.sh/uv). Call it as `uv tool run <tool>`; it behaves the same as `uvx` and survives a broken `uvx` shim.
- The Figma MCP connector is connected and `whoami` works. The user needs a plan that can create FigJam files.
- Apple Silicon (`uname -m` prints `arm64`) for the fast MLX models. Elsewhere, use the faster-whisper fallback in step 2.

## 1. Find the recording

- Use the path the user gives. If none, use the newest `.mp4` or `.mov` in `~/Desktop` (where most screen recorders save by default), and say which file you picked.
- Check it: `ffprobe -v error -show_entries format=duration:stream=codec_type,width,height -of compact "<file>"`. It must have both a video and an audio stream.
- Make a working folder at `<working folder>/<YYYY-MM-DD>-<short-slug>/` (use `<YYYY-MM-DD>-session` until you know the topic, then rename). Put `frames/`, the transcript and `moments.json` there.

## 2. Transcribe

1. Extract audio: `ffmpeg -loglevel error -i "<video>" -vn -ac 1 -ar 16000 audio.wav`
2. Default, Parakeet on MLX (Apple Silicon; fast, accurate English, word and sentence timestamps):
   `uv tool run parakeet-mlx audio.wav --output-format json --output-dir .`
   It writes `audio.json` with `sentences`, each holding word-piece `tokens` with start and end times. Run `uv tool run parakeet-mlx --help` first if the flags have changed. The first run downloads the model from Hugging Face.
3. Fallback, Whisper on MLX:
   `uv tool run --from mlx-whisper mlx_whisper audio.wav --model mlx-community/whisper-small.en-mlx --word-timestamps True --output-format json --output-dir .`
4. Not on Apple Silicon: `uv run --with faster-whisper python` with `WhisperModel("small.en", device="cpu", compute_type="int8")` and `transcribe("audio.wav", word_timestamps=True, vad_filter=True)`. For other languages, use a multilingual model (`small` instead of `small.en`).
5. Normalize whatever the tool wrote into `transcript.json` (segments, each with start, end, text, and words when available) and a readable `transcript.md` with `[mm:ss]` stamps.
6. If transcription fails, stop and tell the user what failed. Do not guess at what was said.

## 3. Find the key moments

Read the whole transcript before choosing. A key moment is one thing the user wants someone to act on or remember:

- **Bug**: something behaves wrong or breaks.
- **Friction**: it works but feels off (confusing, slow, ugly, extra steps).
- **Idea**: a new feature or change the user suggests.
- **Question**: something the user is unsure about and wants decided.
- **Keep**: something the user likes and wants preserved.

Rules:
- Combine a ramble about one thing into one moment. Split a stretch only when the subject changes.
- Skip filler, setup talk ("ok, so now I'm going to open..."), and repeats, unless a repeat adds detail.
- Pick the **frame time** as when the thing is on screen and pointed at. Usually that's the word "this", "here" or "that", or the moment the behavior happens, not when the sentence starts. Use word timestamps when you have them.
- For each moment, record: `n`, `time` (mm:ss), `frame_t` (seconds), `type`, `title` (short, written as a ticket title), `quote` (the user's words, lightly trimmed), `interpretation` (what's wrong or wanted, in plain words), `where` (screen or area of the app), `confidence` (high, or low when you're inferring a lot).
- Print the words with their start times (one line per sentence, `word@seconds`) to find the "this" and "here" moments. It's cheaper than reading the raw JSON.
- If you can't tell what "this" means, look at the frames before deciding. Small previews (`scale=1100:-2`, JPG) are enough to choose from and save context; tile the three candidates side by side with `hstack` to read them in one image. When the commentary talks about behavior ("this is weird", "it jumps"), find where the screen changes with `ffmpeg -i "<video>" -vf "scale=400:-2,select='gt(scene,0.02)',showinfo" -an -f null - 2>&1 | grep -o 'pts_time:[0-9.]*'`. If nothing changes and the frame doesn't explain it, mark the moment low confidence and ask the user in the report.
- Transcription can mishear a word. When a phrase doesn't fit the screen, put your best guess in brackets with a question mark in the quote, and say so in the interpretation.
- Save it all to `moments.json`.
- If the recording has no app feedback (a test, or just talk), say so and still make one moment so the board path gets exercised.

Expect roughly one moment per 30 to 60 seconds of real commentary. If you end up with more than 25, you're probably not combining enough.

## 4. Grab the frames

For each moment:
1. Grab three candidates around `frame_t` (−0.7s, 0, +0.7s):
   `ffmpeg -loglevel error -ss <t> -i "<video>" -frames:v 1 -vf "scale='min(1920,iw)':-2" frames/<n>-<k>.png`
2. Look at the candidates and keep the one that best shows the thing being discussed, with the cursor on it and no motion blur or half-loaded screen. Rename it `frames/<n>.png` and delete the rest.
3. Note what the cursor points at and add it to `where` if it helps. If the frame contradicts your reading of the transcript, fix the interpretation or set `confidence` to low.
4. Keep each image under 10 MB (FigJam's limit). Re-encode as JPG if needed.

## 5. Build the board

1. Load the Figma skills first if they're installed: `figma-create-new-file` (before `create_new_file`) and `figma-use-figjam` (before `use_figma`; pass `skillNames: "figma-use-figjam"`). Load the Figma tool schemas in one ToolSearch `select:` call if they're deferred.
2. Resolve the plan key: from settings, else `whoami` (one plan: use it; several: ask which).
3. `create_new_file` with `editorType: "figjam"`, named `<App> commentary · <YYYY-MM-DD> · <topic>`. Then `create_session` with the new file key, and pass its sessionId on every later Figma call.
4. Use `use_figma` to lay out the board (one call is fine):
   - **Header** at the top left: a bold title text (48px), a meta line (recording date, length, moment count by type), and a wide yellow summary sticky whose first line is `Themes`, followed by 3 to 5 bullets of the main themes.
   - **One cell per moment, in a grid three cells wide**, in time order reading left to right then down: moments 1, 2, 3 on the first row, 4, 5, 6 on the second, and so on. The grid starts 120px below the summary sticky, at x = 0.
   - **Each cell** is a screenshot frame with its sticky to the right. The frame is 960px wide with the screenshot's aspect ratio, **white fill with a dashed gray stroke** (a gray fill disappears on FigJam's gray canvas). Name it `Moment #<n>` and put nothing inside it. The sticky sits at the frame's x + 960 + 64, top-aligned with the frame.
   - **Spacing:** cell width = 960 + 64 + the widest sticky's width (about 1440). Columns are 160px apart, so column c starts at x = c × (cell width + 160). Each row is as tall as its tallest frame or sticky, and rows are 160px apart.
   - **Sticky text**: a wide sticky (`isWideWidth = true`) with this exact text, so it can be read back later:
     ```
     #<n> · <mm:ss> · <Type>
     <title>

     Said: "<quote>"
     Means: <interpretation>
     Where: <where>
     ```
     Add `(low confidence)` after the type when it applies.
   - Sticky colors by type (use `hex/255`): Bug red `#FFB8A8`, Friction orange `#FFD3A8`, Idea blue `#A8DAFF`, Question violet `#D3BDFF`, Keep green `#B3EFBD`.
   - Load the sticky font once from a probe sticky's `text.fontName`. Create every sticky and set its text before placing anything, then place the grid from the stickies' real widths and heights (the script in "Re-laying out the grid" does the placement).
   - Return the placeholder frame node ids, in moment order.
5. `upload_assets` with `count` = number of moments, `nodeIds` = the placeholder ids, `scaleMode: "FIT"`. POST each image to its `submitUrl` as multipart (the filename becomes the layer name):
   `curl -sS -X POST -F "file=@frames/<n>.png;type=image/png" "<submitUrl>"`
   URLs are single-use and expire in 10 minutes, so POST right away. Send at most 60 per call.
6. Check with `get_screenshot` on node `0:1`: every frame has an image, the grid reads 1, 2, 3 across the top, no text overflows, nothing overlaps. Fix and check again.

## 6. Report

Tell the user the board link, the number of moments by type, and one line per moment (`#n mm:ss Type: title`). Point out any low-confidence moments and likely mishearings. Say that they can edit or delete stickies on the board, and then ask you to turn them into tickets. Mention the working folder path. Rename the working folder to its topic slug and delete the candidate frames and previews.

## When the user clarifies a moment

The user often explains a low-confidence moment, or corrects your reading of one, in chat. Update the board straight away. Don't wait until filing.
- When the clarification is about how the app behaves today (what an icon or color means, what a control does) and the app's code is at hand, check the code first, and say in Means what it already does.
- Rewrite the sticky: add the new words to Said as `Later: "<clarification>"`, rewrite Means and Where, drop `(low confidence)`, and change the type and color if they've changed. Find the sticky with `figma.getNodeByIdAsync(id)` or by its `#<n> ·` first line; `page.getNodeById` throws.
- If the sticky grew, re-run the grid placement (see "Re-laying out the grid") so nothing overlaps.
- Update the header's count line and summary sticky if the types or themes changed, and update `moments.json`.
- If two moments turn out to be one problem, say so on both stickies ("Goes with #4") and file them as one ticket later.

## Re-laying out the grid

Run this whenever a sticky changes size, when the user asks to tidy the board, and to convert a board that stacks moments in one column. It finds each frame by its `Moment #<n>` name and each sticky by its `#<n> ·` first line, so it works after the user has edited the board:

```js
const page = figma.currentPage;
const num = s => parseInt(s.replace(/^\D*/, ''));
const frames = page.children.filter(n => n.type === 'FRAME' && n.name.startsWith('Moment #')).sort((a, b) => num(a.name) - num(b.name));
const stickyFor = {};
for (const s of page.children) if (s.type === 'STICKY' && /^#\d+ /.test(s.text.characters)) stickyFor[num(s.text.characters)] = s;
const summary = page.children.find(n => n.type === 'STICKY' && n.text.characters.startsWith('Themes'));
const cellW = 960 + 64 + Math.max(...Object.values(stickyFor).map(s => s.width));
let y = summary.y + summary.height + 120;
for (let r = 0; r * 3 < frames.length; r++) {
  let rowH = 0;
  frames.slice(r * 3, r * 3 + 3).forEach((f, c) => {
    const s = stickyFor[num(f.name)];
    f.x = c * (cellW + 160); f.y = y;
    s.x = f.x + f.width + 64; s.y = y;
    rowH = Math.max(rowH, f.height, s.height);
  });
  y += rowH + 160;
}
```

Moments the user added by hand with no `Moment #<n>` frame are left where they are, so mention them.

## Later: turning notes into tickets

When the user asks for tickets from a board:
1. Read the board with `get_figjam` and parse the stickies. The board is the source of truth, because the user may have edited, deleted or added stickies. Use `moments.json` only to recover the frame files.
2. Resolve the tracker, team and labels from settings. If they're missing, ask which tracker and team, list that team's labels, and propose a mapping. A common one: Bug → `Bug`; Friction and Idea → `Feature` (or `Improvement` for a tweak to something existing); Question → a needs-info label, or ask the user; Keep → no ticket. Add the team's triage label to every ticket if it has one. Follow the project's filing docs when settings name them.
3. Search the tracker for duplicates first, one query per topic, run in parallel. Note the related tickets, especially ones the new feedback would undo (for example, a ticket that added the button the user now wants removed). If the user says they don't understand a control, look up what it does and say so in the ticket before proposing to remove it: that comment is a clarity problem until the user says otherwise.
4. **Group them.** File one parent ticket, `<Area> walkthrough feedback (<YYYY-MM-DD>)`, with only the triage label. Its description holds the board link, 3 to 5 theme bullets, and the Keep moments (Keep gets no ticket of its own). Each moment becomes a sub-issue under the parent. Merge stickies about the same problem into one sub-issue.
5. Show the user the plan as a table (sub-issue title, label, which stickies, related tickets) before filing anything, and file only after a clear yes.
6. Each sub-issue: the title, then **Means**, **Said** (including any `Later:` clarification), **Where**, and **When** (the timestamp plus a markdown link to its sticky, `https://www.figma.com/board/<fileKey>?node-id=<n-m>`). Attach the sticky URL as a link too, and mark the related tickets from step 3 as related.
7. Attach each frame to its sub-issue using the tracker's upload flow.
8. **Link back from the board.** Append `\n\nTicket: <ID>` to each sticky and make the ID a hyperlink with `sticky.text.setRangeHyperlink(start, end, { type: 'URL', value: <issue url> })`. Keep stickies get `Keep, noted in <parent ID>`. Add a linked line under the header's count line: `<Tracker>: <parent ID> <title> (parent of <first ID> to <last ID>)`. Then re-lay out the grid if any sticky grew, update the header counts, and check with `get_screenshot`.
9. Report the parent and each sub-issue as a link with its title, never a bare ID.

### Linear specifics

- Set `parentId` on each sub-issue, pass the sticky URL in `links`, and set `relatedTo` for related tickets.
- **Never write `#N` in Linear text** when the workspace is connected to GitHub: Linear turns "sticky #4" into a link to an unrelated pull request. Write "sticky 4".
- Attach frames one at a time: `prepare_attachment_upload`, then `curl -X PUT --data-binary @frames/<n>.png` with every signed header sent exactly as given, then `create_attachment_from_upload`. The signed URL lasts 60 seconds, so send the PUT, the finalize and the next prepare in one message. A small `put.sh <n> <filename> <url>` script that works out the size and headers saves retyping.

## Running in a cloud container (not recommended)

If you're in a cloud container rather than on the user's computer, two things usually differ:
- huggingface.co may be blocked, so the MLX and faster-whisper models can't download. `sherpa-onnx` (`pip install sherpa-onnx`) with its Whisper and Silero VAD models from the `k2-fsa/sherpa-onnx` GitHub releases (tag `asr-models`) is one way round it.
- The screenshot POST to `mcp.figma.com` is blocked unless that host is in the environment's allowed domains. Without it the board gets stickies but no images. Say so, or suggest running locally instead.
