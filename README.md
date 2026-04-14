# Canvas LMS Scraper

Exports Canvas course modules to self-contained offline HTML files. Uses your browser session cookie — no Canvas API key required. Discussions, assignments, quizzes, pages, and files are all handled automatically in a single run.

---

## Requirements

**Python 3.10+**

```bash
pip install requests beautifulsoup4 lxml tqdm playwright
playwright install chromium
```

---

## Usage

```bash
python canvas_scraper.py \
  --course-url https://canvas.school.edu/courses/12345 \
  --session "_csrf_token=abc; canvas_session=xyz"
```

### All flags

| Flag | Required | Default | Description |
|---|---|---|---|
| `--course-url` | ✅ | — | Course root URL, e.g. `https://canvas.school.edu/courses/12345` |
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

> The script validates the cookie on startup and exits immediately if redirected to the login page.

---

## What happens when you run it

The script fetches the course's Modules page and presents a numbered list:

```
============================================================
Available Modules:
  [1] Week 1 — Introduction
  [2] Week 2 — The Civil War
  [3] Week 3 — Reconstruction
============================================================
Enter a comma-separated list of module numbers to scrape (or 'all'/Enter to scrape everything):
```

Enter numbers like `1,3` to scrape specific modules, or press Enter / type `all` for everything. Each selected module is then processed item by item.

---

## How each item type is handled

### Pages and assignments

Plain HTML pages are fetched with `requests` and saved directly. Assignment pages are rendered entirely by JavaScript — when the script detects an empty React shell, it automatically launches a headless Chromium browser (Playwright) to load the page and extract a clean card showing:

- Due date
- Points / grade
- Attempts allowed
- Status (Missing / Late / Submitted / Graded)
- Assignment description (if one exists)

> **Note on assignment descriptions:** Canvas controls whether students can see their submission responses and scores via quiz/assignment result settings. The exported output reflects exactly what your account can see at the time of scraping.

### Quizzes

Exported with colour-coded results — correct answers highlighted green, wrong selections red. What appears in the export depends on the instructor's quiz settings ("Let Students See Their Responses" / "Let Students See the Correct Answers").

### Discussions

Discussion items inside a module are automatically scraped with the full Playwright headless browser. You will be prompted to select one or more participants by number (comma-separated):

```
============================================================
Participants found in this Discussion:
  [1] Alice Johnson
  [2] Bob Smith
  [3] Instructor Name
============================================================
Enter comma-separated numbers of the people whose replies you want to scrape:
```

Each selected person gets their own HTML file containing the discussion prompt followed by all of their replies with avatars and timestamps.

### Files

Downloaded directly into the module's `attachments/` folder. A stub HTML page is written so the module listing stays complete.

### External links and tools

All external URLs and LTI tool links found in a module are collected into a single text file at `attachments/external_links.txt`, one entry per link (title on one line, URL on the next).

---

## Output structure

```
Canvas_Export/
└── Modules/
    ├── Week 1 — Introduction/
    │   ├── 01 - Syllabus.html
    │   ├── 02 - Reading Notes.html
    │   ├── 03 - Introduction Discussion/
    │   │   ├── Discussions/
    │   │   │   └── Introduction Discussion/
    │   │   │       ├── replies_Alice Johnson.html
    │   │   │       ├── media/
    │   │   │       └── attachments/
    │   ├── media/
    │   └── attachments/
    │       ├── lecture_slides.pdf
    │       └── external_links.txt
    └── Week 2 — The Civil War/
        ├── 01 - Lecture Slides.html
        ├── 02 - Quiz American Yawp Ch 15.html
        └── attachments/
            └── external_links.txt
```

Every HTML file is self-contained with embedded CSS — readable in any browser with no internet connection.

---

## Resume support

A `.scraper_state.json` file is written inside the output directory to track completed items. If the run is interrupted (Ctrl-C or a network error), re-running the same command skips already-finished items and picks up where it left off.

---

## Notes

- **Sessions expire.** Canvas cookies typically last a few hours. If you see an authentication error mid-run, grab a fresh cookie and re-run — completed items are skipped.
- **Windows path limits.** Module and item names are capped at 55 and 60 characters respectively to stay within Windows' 260-character MAX_PATH limit.
- **Media downloads are best-effort.** Canvas Studio / Arc video embeds are attempted through multiple resolution strategies. Inaccessible media is replaced with a visible warning box rather than silently dropped.
- **No API calls are made.** Everything is scraped from the same HTML your browser sees, so it respects whatever visibility your account has.
