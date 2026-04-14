# Canvas LMS Scraper

A command-line tool that exports Canvas course content to self-contained offline HTML files. It uses your browser session cookie for authentication — no Canvas API key required.

---

## Requirements

**Python 3.10+**

Install dependencies:

```bash
pip install requests beautifulsoup4 lxml tqdm playwright
playwright install chromium
```

---

## Usage

For `--mode modules`:
```bash
python canvas_scraper.py \
  --course-url https://canvas.school.edu/courses/12345 \
  --mode modules \
  --session "_csrf_token=abc; canvas_session=xyz"
```

For `--mode discussions` or `--mode both`, pass the specific discussion thread URL:
```bash
python canvas_scraper.py \
  --course-url https://canvas.school.edu/courses/12345/discussion_topics/67890 \
  --mode discussions \
  --session "_csrf_token=abc; canvas_session=xyz"
```

> **Note:** `--mode both` scrapes one discussion thread and all modules in a single run. Because `--course-url` is a single argument, only one discussion thread can be targeted per run.

### All flags

| Flag | Required | Default | Description |
|---|---|---|---|
| `--course-url` | ✅ | — | Course root URL (`/courses/ID`) for modules; specific discussion thread URL (`/courses/ID/discussion_topics/ID`) for discussions or both |
| `--mode` | ✅ | — | What to scrape: `discussions`, `modules`, or `both` |
| `--session` | ✅ | — | Raw cookie string from your browser (see below) |
| `--output` | | `Canvas_Export` | Directory to write files into |
| `--zip` | | off | Zip the output folder when finished |
| `--retries` | | `3` | Max HTTP retries per request |
| `--debug` | | off | Enable verbose DEBUG logging |

---

## How to get your session cookie

1. Log in to Canvas in your browser.
2. Open DevTools → **Application** (Chrome) or **Storage** (Firefox) → **Cookies**.
3. Copy the values for `_csrf_token` and `canvas_session`.
4. Pass them as a single quoted string:

```
"_csrf_token=abc123; canvas_session=xyz789"
```

> The script validates the cookie on startup and exits immediately if you have been redirected to the login page.

---

## Modes

### `--mode modules`

Fetches the course's Modules page and lists every module interactively:

```
============================================================
Available Modules:
  [1] Week 1 — Introduction
  [2] Week 2 — The Civil War
  [3] Week 3 — Reconstruction
============================================================
Enter a comma-separated list of module numbers to scrape (or 'all'/Enter to scrape everything):
```

Enter numbers like `1,3` to scrape specific modules, or press Enter / type `all` to scrape everything.

Each module item is saved as a standalone HTML file with all images, videos, and attachments downloaded locally alongside it. Supported item types: pages, assignments, quizzes, discussions, and files.

#### Quiz output

Quizzes are exported with colour-coded results — correct answers highlighted in green, wrong selections in red. **The content of the exported quiz depends entirely on what Canvas shows your account.** Specifically:

- If the quiz's **"Let Students See Their Responses"** setting is enabled, your selected answers will be visible in the export.
- If **"Let Students See the Correct Answers"** is also enabled, the correct answers will be shown alongside your selections.
- If neither setting is enabled, or the quiz window has closed, the export will only contain the questions and whatever Canvas renders for your account at the time of scraping.

**Output structure:**

```
Canvas_Export/
└── Modules/
    ├── Week 1 — Introduction/
    │   ├── 01 - Syllabus.html
    │   ├── 02 - Reading Notes.html
    │   ├── media/
    │   └── attachments/
    └── Week 2 — The Civil War/
        ├── 01 - Lecture Slides.html
        ├── 02 - Quiz American Yawp Ch 15.html
        ...
```

---

### `--mode discussions`

Loads the discussion thread in a headless Chromium browser, automatically handles multi-page threads by clicking through all pagination buttons, then lists every participant:

```
============================================================
Participants found in this Discussion:
  [1] Alice Johnson
  [2] Bob Smith
  [3] Instructor Name
============================================================
Enter comma-separated numbers of the people whose replies you want to scrape:
```

Select one person by number. The script collects every reply that person made across all pages and writes a single clean HTML file containing only their posts, with avatars, timestamps, and embedded media.

**Output structure:**

```
Canvas_Export/
└── Discussions/
    └── GROUP DISCUSSION Introduce Yourselves Please/
        ├── replies_Alice Johnson.html
        ├── media/
        └── attachments/
```

---

### `--mode both`

Runs the discussions scraper first (using the thread URL provided via `--course-url`), then the modules scraper (navigating to the course's modules page automatically). Both interactive prompts appear in sequence.

```bash
python canvas_scraper.py \
  --course-url https://canvas.school.edu/courses/12345/discussion_topics/67890 \
  --mode both \
  --session "_csrf_token=abc; canvas_session=xyz"
```

---

## Resume support

The scraper writes a `.scraper_state.json` file inside the output directory to track completed items. If the run is interrupted (Ctrl-C or network error), re-running the same command will skip already-finished items and pick up where it left off.

---

## Output format

Every exported page is a self-contained HTML file with:

- Embedded CSS — readable in any browser with no internet connection.
- Locally downloaded images and videos (saved to a `media/` subfolder).
- Locally downloaded file attachments (saved to an `attachments/` subfolder).
- Quiz results styled with colour-coded correct/incorrect answers (subject to quiz result visibility settings — see above).
- Discussion threads formatted with author avatars and timestamps.

---

## Notes

- **Sessions expire.** Canvas session cookies typically last a few hours. If you see an authentication error mid-run, grab a fresh cookie and re-run — completed items will be skipped.
- **Windows path limits.** Module and item names are capped at 55 and 60 characters respectively to stay within Windows' 260-character MAX_PATH limit.
- **Media downloads are best-effort.** Canvas Studio / Arc video embeds are attempted through multiple resolution strategies; inaccessible media is replaced with a visible warning box rather than silently skipped.
- **No API calls are made.** Everything is scraped from the same HTML your browser sees, so it respects whatever visibility your account has.
