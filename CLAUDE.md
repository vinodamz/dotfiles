# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projects Overview

This home directory contains three independent Python utility projects:

- **KreedoCode** (`~/KreedoCode/`) — Bulk-fetches Kreedo activity data by ID range, saves JSON responses, and generates structured questions from educational content.
- **KreedoDataFetcher** (`~/KreedoDataFetcher/`) — CLI tool to authenticate with Kreedo's REST API and export child/activity records to Excel.
- **VideoFrane** (`~/VideoFrane/`) — Video processing pipeline: extract frames → deduplicate → remove watermarks → convert to PowerPoint.

---

## KreedoCode

**Python version:** 3.11 (virtualenv at `venv/`)

**Setup:**
```bash
cd KreedoCode
source venv/bin/activate
pip install -r requirements.txt
```

**Run:**
```bash
# Fetch activity JSON responses for a range of IDs
python kreedo_fetch.py  # edit START_ACTIVITY_ID / END_ACTIVITY_ID at the bottom of the file
```

**Architecture:**
- `kreedo_fetch.py` — iterates an ID range against `https://6t.kreedo.solutions/api/activity/activity_detail_mob/{id}`, saves each response to `kreedo_responses/<id>/<id>_response.json`, logs status to `kreedo_log.csv`. JWT token is hardcoded in `HEADERS` and expires; update it when you get 401 responses.
- `Book_question.py` — reads activity JSON and generates structured questions.
- `kreedo_responses/` — cached API responses (18 000+ folders); do not delete.

---

## KreedoDataFetcher

**Python version:** 3.12 (virtualenv at `.venv/`)

**Setup:**
```bash
cd KreedoDataFetcher
source .venv/bin/activate
pip install requests pandas openpyxl
```

**Run:**
```bash
# Login with token, fetch children for a school
python main.py --url <base_url> --token <jwt_token> --fetch-children --school-id <id>

# Login with credentials (token is cached in .token file)
python main.py --url <base_url> --credentials <user_id> <password> --fetch-children --school-id <id>

# Fetch activities (reads child/children.xlsx produced above)
python main.py --url <base_url> --token <jwt_token> --fetch-activities --child-name all
```

**Architecture:**
- `auth.py` — `KreedoAuth` wraps a `requests.Session` with `Authorization: JWT <token>`. Supports token or credential login; credentials hit `POST /users/login`. Token is validated against `GET /users/logged_in_user_detail`.
- `child_service.py` — two-phase fetch: list children via `GET /child/child_list_create` (paginated), then enrich each child with `GET /child/child_retrive_update_delete/{id}` and subjects via `POST /plan/subject_list_by_child`. Exports to `child/children.xlsx` (sheets: Children, Parents, Subjects). Activities are fetched from `POST /activity/flag_based_activity` and written to `child/child_activities.xlsx` with one sheet per child.
- `main.py` — argparse CLI; token is persisted to `.token` and reused across runs.

The API response may nest data under a `"data"` key; `child_service.py` handles both flat and nested response shapes.

---

## VideoFrane

**Python version:** 3.12 (virtualenv at `.venv/`)

**Setup:**
```bash
cd VideoFrane
source .venv/bin/activate
pip install opencv-python Pillow imagehash python-pptx
```

**Run each stage independently:**
```bash
# 1. Extract frames from video
python video_to_frame.py <video_path> [output_dir] [--save]

# 2. Deduplicate frames (reads frames/, writes unique frames to cleaned/)
python deduplicate_frames.py

# 3. Remove watermarks (reads frames or cleaned/)
python remove_watermark.py

# 4. Convert frames to PowerPoint (widescreen 16:9)
python frames_to_ppt.py
```

**Architecture:**
- Each script is a standalone stage in the pipeline. They share directory conventions: `frames/` for raw extracted frames, `cleaned/` for deduplicated output.
- Deduplication uses perceptual hashing (`imagehash`) to skip visually identical frames.
- `frames_to_ppt.py` generates `output.pptx`; the existing file is ~197 MB.
