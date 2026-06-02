# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Memory File

Persistent memory is stored at `~/.claude/projects/-Users-vinodamz/memory/MEMORY.md`. Update it when project structure, credentials, or key conventions change.

## Projects Overview

- **MontessoriTraineeTeacher** (`~/MontessoriTraineeTeacher/`) — Unified PHP+MySQL app combining the Trainee Teacher Assessment (formerly MTT) and the Task Manager (formerly LGTaskManager) into one login, one users table, two top-level modules (`assessment/` + `tasks/`). Live at https://mtt.thelittlegraduates.in/. → https://github.com/vinodamz/MontessoriTraineeTeacher
- **KreedoCode** (`~/KreedoCode/`) — Bulk-fetches Kreedo activity data by ID range, saves JSON responses, and generates structured questions from educational content. → https://github.com/vinodamz/KreedoCode
- **KreedoDataFetcher** (`~/KreedoDataFetcher/`) — CLI tool to authenticate with Kreedo's REST API and export child/activity records to Excel. → https://github.com/vinodamz/KreedoDataFetcher
- **VideoFrane** (`~/VideoFrane/`) — Video processing pipeline: extract frames → deduplicate → remove watermarks → convert to PowerPoint. → https://github.com/vinodamz/VideoFrane

> **Note:** `~/LGTaskManager/` and github.com/vinodamz/LGTaskManager still exist on disk and on GitHub but are **superseded** — all task-manager work now lives under `~/MontessoriTraineeTeacher/tasks/`. Treat that repo as a historical artifact.

---

## MontessoriTraineeTeacher (unified app)

Single PHP+MySQL app combining the **Trainee Teacher Assessment** (formerly MTT) and the **Task Manager** (formerly LGTaskManager). One PIN-based login, one users table, role + per-module access. Live and serving at https://mtt.thelittlegraduates.in/.

**Repo:** https://github.com/vinodamz/MontessoriTraineeTeacher (kept the name, will rename later)
**Local path:** `~/MontessoriTraineeTeacher/`
**Stack:** PHP 8.1+ / MySQL InnoDB utf8mb4 — server-rendered HTML, no build step
**Hosting:** Hostgator cPanel (account `ideyyfbn`); docroot `/home/ideyyfbn/thelittlegraduates.in/mtt/`

**File layout:**
```
~/MontessoriTraineeTeacher/
├── index.php             # Unified home / module picker (single-module users auto-redirect)
├── login.php             # PIN landing — one login for everyone
├── logout.php
├── admin.php             # Central user mgmt (role + per-module access)
├── install.php           # First-admin bootstrap (delete after use)
├── migrate.php           # Schema migrator — admin-only steady state; bootstrap mode when users table missing/legacy
├── includes/             # auth.php, db.php, functions.php (union of both apps' helpers),
│                         # header.php, footer.php, config.example.php
├── assessment/           # Former MTT pages: index.php, assess.php, progress.php,
│                         # baseline.php, custom_indicators.php, admin.php
├── tasks/                # Former LGTaskManager pages: index.php, tasks.php, calendar.php,
│                         # admin.php, debug-recurrences.php, reset-opcache.php
├── assets/css/           # style.css (MTT base, wins cascade) + tasks.css (LG selectors, loaded first)
├── assets/js/            # login.js, assess.js, kanban.js
└── sql/                  # schema.sql (fresh-DB unified schema),
                          # migrate_001_unify_users.sql (in-place upgrade), seeds.sql
```

**Auth + data model:**
- One `users` table with `role ENUM('teacher','admin')` and `modules SET('tasks','montessori')`. Admins implicitly access all modules.
- Sessions store `user_id` / `user_name` / `user_role` / `user_modules`.
- `current_user()` / `require_login()` / `require_admin()` / `require_module($name)` live in `includes/auth.php`.
- PINs: bcrypt-hashed, rate-limited (5 tries → 30 s lock), 4–6 digit numeric.
- The old `teachers` table no longer exists — its rows were merged into `users` with IDs preserved. Every FK in `students` / `evaluation_cards` / `assessments` / `assessment_comments` / `student_baselines` / `student_custom_indicators` now points at `users(id)` instead of `teachers(id)`.

**Live DB snapshot (after migrate_001_unify_users.sql ran on 2026-05-18):**
- 6 users (4 teachers + 2 admins, IDs preserved from old `teachers`)
- 41 students, 443 evaluation_cards, 132 assessments, 37 assessment_comments, 40 student_baselines (all intact)
- 119 skill_indicators, 6 rating_config rows, 0 student_custom_indicators
- task_columns seeded with default 3 (To do / In progress / Done); task_recurrences + tasks empty
- Stray LGTaskManager `users` / `tasks` / `task_columns` / `task_recurrences` tables from an earlier LG deployment against the MTT database were dropped via `/migrate.php?confirm=drop-legacy-lg` before the unification migration ran.

**Deploy pipeline** (identical to the old MTT + LGTaskManager flow):
GitHub Actions `php -l` lint → cPanel UAPI `VersionControl/update` → `VersionControlDeployment/create` runs `.cpanel.yml` rsync. No build, no deploy branch.
- cPanel repo clone path: `/home/ideyyfbn/repos/MontessoriTraineeTeacher` tracking `main`
- Required GitHub secrets: `CPANEL_HOST`, `CPANEL_USER`, `CPANEL_TOKEN`
- `includes/config.php` is gitignored and excluded from rsync — the live server keeps its own copy with DB credentials

**Gotchas (battle-scarred):**
1. **Hostgator file-permission gotcha** — `.cpanel.yml` MUST chmod 755 the docroot + parent, then `find -type d -exec chmod 755 {} +` and `find -type f -exec chmod 644 {} +` after rsync, or every URL 404s. Already handled in the current `.cpanel.yml`.
2. **Live `includes/config.php` is out of sync with the example** — `includes/config.example.php` is now branded as "Little Graduates" with `'session_name' => 'LG_SESSION'`, but the live server's `config.php` was not overwritten by deploy. So the live session cookie is still `MTT_SESSION` and the visible app name is "Trainee Teacher Assessment" until someone edits that file on the server.
3. **CSS conflict on .pin-overlay** — `tasks.css` (LG) sets `.pin-overlay { opacity: 0 }` for an `.is-open` class-driven fade-in animation that the unified `login.js` doesn't use. `style.css` overrides with `opacity: 1` and `transform: none` on `.pin-overlay` / `.pin-modal`. If a modal-like component ever shows up "click does nothing", check computed opacity first.
4. **`migrate.php` has a bootstrap mode** — when `users_table_state()` returns `'missing'` or `'legacy'`, the page bypasses `require_admin()` because `login.php` can't query the DB yet. Once state is `'unified'`, `/migrate.php` reverts to admin-only. Intentional; gated by an idempotent state check, not a feature flag.

---

## KreedoCode

**Repo:** https://github.com/vinodamz/KreedoCode

**Python version:** 3.11 (virtualenv at `venv/`)

**Setup:**
```bash
cd KreedoCode
source venv/bin/activate
pip install -r requirements.txt
# For Book_question.py (Streamlit app) — install separately:
pip install streamlit langchain langchain-openai pydantic
# For process_live_sheet.py (Google Sheets):
pip install google-api-python-client google-auth
```

**Run:**
```bash
# 1. Generate a fetch schedule splitting ID range across N days
python create_schedule.py  # edit TOTAL_START_ID, TOTAL_END_ID, NUM_DAYS at the bottom; outputs schedule.json

# 2a. Fetch a specific ID range directly
python kreedo_fetch.py  # edit START_ACTIVITY_ID / END_ACTIVITY_ID at the bottom

# 2b. Fetch using the schedule (processes all days, skips already-downloaded IDs)
python daily_fetcher.py

# 3. Convert cached JSON responses into relational DataFrames (prints to stdout as markdown tables)
python json_to_table.py

# 4. Run the Streamlit question-generation app (requires OPENAI_API_KEY)
streamlit run Book_question.py

# 5. Read from Google Sheets and find activities per child
python process_live_sheet.py  # requires little-graduates-3663b17fa316.json service account file
```

**Architecture:**
- `kreedo_fetch.py` — iterates a hardcoded ID range against `https://6t.kreedo.solutions/api/activity/activity_detail_mob/{id}`, saves each response to `kreedo_responses/<id>/<id>_response.json`, logs to `kreedo_log.csv`. JWT token is hardcoded in `HEADERS`; update it on 401 errors.
- `create_schedule.py` — `ActivityScheduler` class: shuffles the full ID range and splits into N daily chunks, saves to `schedule.json`.
- `daily_fetcher.py` — reads `schedule.json`, iterates all days calling the same fetch logic as `kreedo_fetch.py`, skips already-downloaded IDs (checks for existing file before making the request).
- `json_to_table.py` — scans all `kreedo_responses/**/*_response.json` files and builds relational DataFrames: `activities`, `materials`, `assets`, `activity_material_link`, `activity_asset_link`. Deduplicates materials and assets by ID.
- `Book_question.py` — Streamlit app: upload PDFs → select question type (Short/Long Descriptive, MCQ, True/False, Word-Puzzle) → calls `gpt-4o-mini` via LangChain with a Pydantic structured output schema → displays results as a dataframe. `OPENAI_API_KEY` is set inline (replace the placeholder `*********`).
- `process_live_sheet.py` — `ActivityManager` orchestrates `GoogleSheetsClient` (service account auth) → `ActivityDataProcessor` (reads two sheets, splits multi-activity cells on `$$` or `,`, merges on `LG Activity ID`) → `ChildActivityFinder` (filters by child name + shared group names like `pg`, `nur`, `lkg`, `ukg`). Spreadsheet ID and sheet names are hardcoded in `main()`.
- `kreedo_responses/` — 18 000+ cached response folders; do not delete.

---

## KreedoDataFetcher

**Repo:** https://github.com/vinodamz/KreedoDataFetcher

**Python version:** 3.12 (virtualenv at `.venv/`)

**Setup:**
```bash
cd KreedoDataFetcher
source .venv/bin/activate
pip install -r requirements.txt  # requests==2.31.0, pandas, openpyxl
```

**Run:**
```bash
# Login with token, fetch children for a school
python main.py --url <base_url> --token <jwt_token> --fetch-children --school-id <id>

# Login with credentials (token cached to .token for reuse)
python main.py --url <base_url> --credentials <user_id> <password> --fetch-children --school-id <id>

# Fetch completed activities for all children (reads child/children.xlsx)
python main.py --url <base_url> --token <jwt_token> --fetch-activities --child-name all

# Fetch activities for a specific child by name (partial match)
python main.py --url <base_url> --token <jwt_token> --fetch-activities --child-name "John"

# Debug: inspect raw API responses for subjects and activities
python inspect_response.py  # reads .token and child/children.xlsx directly
```

**Architecture:**
- `auth.py` — `KreedoAuth` wraps a `requests.Session` with `Authorization: JWT <token>`. Credential login hits `POST /users/login` and extracts token from `data.token` or `token`/`access_token` keys. Token validity is checked via `GET /users/logged_in_user_detail`.
- `child_service.py` — two-phase fetch:
  1. **Children**: paginated `GET /child/child_list_create` (limit=100), then per-child `GET /child/child_retrive_update_delete/{id}` + `POST /plan/subject_list_by_child` (needs `academic_session`, `child`, `user_id` decoded from JWT payload). Exports to `child/children.xlsx` with sheets: Children, Parents, Subjects.
  2. **Activities**: reads `child/children.xlsx`, groups by child, fetches `POST /activity/flag_based_activity` (flag=`completed`, paginated limit=50). Exports to `child/child_activities.xlsx` with one sheet per child (sheet name sanitised to ≤30 chars).
- `main.py` — argparse CLI; `--fetch-children` requires `--school-id`; `--fetch-activities` prompts for child name if `--child-name` not provided (defaults to `all` in non-interactive mode).
- `inspect_response.py` — standalone debug script; reads `.token`, loads the first child from Excel, and prints raw subject + activity API responses.

The API response may nest data under a `"data"` key; all service functions handle both flat and nested shapes. `user_id` is decoded from the JWT payload (base64, middle segment) and is required for subject fetching.

---

## VideoFrane

**Repo:** https://github.com/vinodamz/VideoFrane

**Python version:** 3.12 (virtualenv at `.venv/`)

**Setup:**
```bash
cd VideoFrane
source .venv/bin/activate
pip install opencv-python Pillow imagehash python-pptx
```

**Pipeline — run stages in order:**
```bash
# 1. Extract frames from video (--save writes frame_XXXXXX.jpg to output_dir)
python video_to_frame.py <video_path> [output_dir] [--save]

# 2. Deduplicate frames in-place (deletes duplicates from the same folder)
python deduplicate_frames.py [frames_dir]              # default dir: frames
python deduplicate_frames.py frames --threshold 8      # looser similarity (default: 5)
python deduplicate_frames.py frames --dry-run          # preview without deleting

# 3. Remove watermark — interactive (draw box on first image) or batch with known coords
python remove_watermark.py <image_or_folder>                          # interactive
python remove_watermark.py frames --coords 10 850 200 40              # batch, TELEA algo
python remove_watermark.py frames --coords 10 850 200 40 --algo ns    # Navier-Stokes
python remove_watermark.py frames --coords 10 850 200 40 --out cleaned --radius 10

# 4. Convert images folder to PowerPoint (widescreen 16:9, black background)
python frames_to_ppt.py [folder] [output.pptx]
python frames_to_ppt.py frames output.pptx --title "My Video"
python frames_to_ppt.py frames output.pptx --captions --bg-color 1a1a2e
```

**Architecture:**

- `video_to_frame.py` — opens video with OpenCV, prints metadata (resolution, FPS, duration), saves frames as `frame_XXXXXX.jpg` when `--save` is passed. Without `--save` it only reads and counts frames.

- `deduplicate_frames.py` — computes perceptual hash (`imagehash.phash`, 8×8 DCT) for each JPEG, then compares **consecutive frames** (not all-pairs): if the hash distance ≤ threshold the later frame is deleted. Operates **in-place** on the input directory — it does not write to a separate folder.

- `remove_watermark.py` — two modes:
  - *Interactive*: opens the first image via `cv2.selectROI`, user draws a bounding box, prints the resulting `--coords` for reuse in batch runs.
  - *Batch*: applies the same `(x, y, w, h)` region to every image. Builds a binary mask (with 2 px padding), then calls `cv2.inpaint` (TELEA fast or Navier-Stokes). Saves results to `--out` (default `cleaned/`).

- `frames_to_ppt.py` — collects images from folder (sorted), fits each to 16:9 (13.33 × 7.5 in) with letterbox/pillarbox aspect-ratio preservation, adds as a picture on a blank slide with a solid background colour. Optional `--title` adds a title slide first; `--captions` adds filename text at the bottom of each slide. Output is `output.pptx` (existing file ~197 MB).

**Typical full workflow:**
```
video_to_frame.py → frames/          (raw frames)
deduplicate_frames.py frames         (removes near-duplicates in-place)
remove_watermark.py frames --coords X Y W H --out cleaned   (writes to cleaned/)
frames_to_ppt.py cleaned output.pptx
```
