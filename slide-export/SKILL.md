---
name: slide-export
version: 1.0.0
description: "Export HTML slides (Reveal.js or any HTML presentation) to PDF at the correct aspect ratio using decktape"
---
# /slide-export — Export HTML Slides to PDF

Export any HTML presentation to a properly sized PDF using decktape. Works with Reveal.js, generic HTML slides, and most other frameworks. Serves slides over a local HTTP server so fonts, images, and external assets load correctly.

## Usage

```
/slide-export [path/to/slides.html]
```

If no path is passed, ask for the HTML file path before proceeding.

## Instructions

### Step 1: Pre-flight

Check that decktape is installed:
```bash
which decktape
```

If not found, tell the user and stop:
> "decktape is required. Install it with:
> - Mac: `brew install decktape` or `npm install -g decktape`
> - Other: `npm install -g decktape`
> Then re-run `/slide-export`."

### Step 2: Locate the file

Use the argument as the HTML file path. If no argument was given, ask: "What's the path to the HTML file?"

Resolve to an absolute path. Confirm it exists:
```bash
ls "[absolute_path]"
```

If it doesn't exist, stop and tell the user.

### Step 3: Detect slide dimensions

Read the HTML file. Look for explicit dimensions in this order:

1. **Reveal.js config** — search for `width:` and `height:` inside a `Reveal.initialize({...})` block. Extract both integer values.
2. **CSS on slide container** — look for explicit `width` and `height` pixel values on `.slides`, `.slide-deck`, or `section.present`.
3. **Viewport meta tag** — check `<meta name="viewport">` for width/height hints.

If found, use those values as `WIDTHxHEIGHT`.

If not found, ask: "What size are your slides? Common options: `1920x1080` (16:9 HD), `1280x720` (16:9), `960x700` (Reveal.js default), or enter a custom `WIDTHxHEIGHT`."

Default if the user skips: **1920x1080**.

### Step 4: Start a local HTTP server

Serving over HTTP ensures fonts, images, and web-hosted assets render correctly.

Get the directory containing the HTML file. Start Python's built-in server on port 8989:
```bash
python3 -m http.server 8989 --directory "[html_directory]" &
echo $!
```

Capture the PID. Sleep 1 second to let it start.

If port 8989 is in use, try 8990, then 8991.

### Step 5: Run decktape

Output path: same directory as the HTML, same filename with `.pdf` extension.

```bash
decktape automatic \
  -s [WIDTH]x[HEIGHT] \
  --load-pause 500 \
  -p 100 \
  http://localhost:[PORT]/[filename] \
  "[output_path]"
```

Flag rationale:
- `automatic` — auto-detects the slide framework (Reveal.js, Remark, etc.)
- `--load-pause 500` — waits 500ms after page load before capturing (lets fonts and images settle)
- `-p 100` — 100ms pause between slides (fast but not choppy)

If decktape fails, show the raw error. If it looks like a framework detection issue, suggest the user retry with `decktape reveal` instead of `automatic`.

### Step 6: Kill the server

```bash
kill [SERVER_PID] 2>/dev/null
```

### Step 7: Confirm

Tell the user the PDF path and open it:
```bash
open "[output_path]"
```

## Notes

- **Animated slides:** If transitions don't fully render, suggest increasing the pause: re-run with `-p 500` or higher.
- **Custom fonts:** If fonts look wrong in the PDF, the slides likely rely on Google Fonts or another CDN. The HTTP server handles this correctly; `file://` would not.
- **Reveal.js specifically:** If `automatic` fails on a Reveal.js deck, use `decktape reveal` instead.
- **Output location:** By default the PDF lands next to the HTML file. If the user wants it somewhere else, ask before running.
- **Large decks:** decktape processes one slide at a time — large decks (100+ slides) may take a minute.
