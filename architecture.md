# Architecture: chesspipe

## 1. Overview

chesspipe is a five-stage Python pipeline that extracts chess diagrams from
scanned book pages, converts them to FEN notation using the Claude Vision API,
allows human correction of uncertain positions, evaluates each position with
Stockfish, and produces an HTML report flagging positions where the book's
recommended move disagrees significantly with modern engine analysis.

The pipeline processes two distinct book formats: Book A, with a fixed grid
layout and a separate answers scan, and Book B, with diagrams embedded in
flowing text columns. The design is deliberately incremental: Phase 1 delivers
the core pipeline without move comparison; Phase 2 adds descriptive-notation
parsing for Book A; Phase 3 adds flow-layout board detection for Book B.

---

## 2. Critique of the README Architecture

The following issues were found in the README. They are resolved in this
document. The developer should treat this architecture as authoritative where
it conflicts with the README.

### 2.1 Book A Scans Are Stored Rotated 90° Clockwise

The README says "3-row × 2-column grid," which is correct for the book's
natural portrait orientation. However, the page scans are stored sideways:
the image file is in landscape orientation, rotated 90° CW from portrait. The
text labels ("BLACK MOVES FIRST", "96 · The OVERWORKED PIECE") run along the
edges and are only legible when the image is rotated back.

In the stored landscape image the diagrams appear as 2 rows × 3 columns, but
after rotating 90° CW to portrait they form the correct 3 rows × 2 columns:

```
Portrait (text upright, after rotation):
┌──────────┬──────────┐
│   463    │   464    │  ← row 0
├──────────┼──────────┤
│   465    │   466    │  ← row 1
├──────────┼──────────┤
│   467    │   468    │  ← row 2
└──────────┴──────────┘
  col 0      col 1
  (odd)      (even)
```

The numbering follows standard reading order (left-to-right, top-to-bottom)
with odd numbers in the left column and even numbers in the right column.
The formula is: `diagram_number = N + 2*row + col`, where N is the lowest
diagram number on the page and col is 0 (left) or 1 (right).

The extractor must rotate each page image 90° CW before slicing the grid
(see §6.1, step 3).

### 2.2 Side-to-Move Is Unaddressed

FEN requires a side-to-move field (`w` or `b`). Diagrams do not encode this,
but scanned pages do: the header "BLACK MOVES FIRST" appears on the diagram
page and applies to all six diagrams on that page. The architecture adds a
per-page side-to-move field that is captured during extraction and stored in
the database alongside each diagram record.

### 2.3 The Answers Scan Is a Real Problem

The answers scan (`462-478.jpg`) is dense multi-column text with descriptive
notation embedded in explanatory prose. "OCR or manual transcription" is not a
design. This architecture adds a dedicated **Stage 1b — Answers Extraction**
(`answers.py`) that sends cropped answer-column images to the Claude Vision API
with a structured prompt, returning a JSON mapping of diagram number → first
move in descriptive notation.

### 2.4 Data Layout Mismatch

The README shows `data/pages/book_a/` as the input directory. The actual data
lives at `data/book_a/`. This architecture uses the actual layout.

### 2.5 Flat JSON Files Are the Wrong Data Store

Two mutable JSON files (`positions.json`, `analysis.json`) shared across
multiple stages invite corruption on crash and make resume impossible. This
architecture uses a single **SQLite database** (`data/pipeline.db`) as the
shared state store. Each stage reads its inputs from and writes its outputs to
the database. Stages are fully idempotent: re-running a stage skips records
that are already in the target state.

### 2.6 No Idempotency for the Claude API

With ~900 images for Book A, re-running Stage 2 after a failure re-bills every
image already processed. All Claude API calls in this architecture check the
database before calling the API and skip records that already have a result.

### 2.7 Review UI Is Unspecified

The README says `review.py` displays images and allows FEN editing, but names
no UI mechanism. This architecture specifies a **local Flask web application**
running on `localhost:5000` that serves one position at a time. See §6.3.

### 2.8 Reinforcement-Learning Aside Removed

The paragraph in the README about regenerating diagrams, training a critic, and
applying reinforcement learning is brainstorming, not architecture. It is not
included in this design.

### 2.9 Stockfish Wrapper Package Removed

The README lists the `stockfish` PyPI package as optional while simultaneously
saying `python-chess` is used directly. `python-chess` provides its own UCI
engine interface; the `stockfish` wrapper is not needed and is omitted from
the dependency list.

### 2.10 Phase 2 Integration Made Explicit

Phase 2 (descriptive notation parsing) is now a named stage (Stage 1b) that
runs on Book A's answers scan. Its output is stored in the database and
consumed by Stage 4 (Stockfish analysis), which compares the book move against
the engine's best move.

---

## 3. Constraints and Assumptions

**Constraints**
- Python 3.11+
- Stockfish binary built from source; path specified in `config.yaml`
- Claude API key in environment variable `ANTHROPIC_API_KEY`
- Input images are individual PNG or JPG files, already scanned; no PDF handling
- Book A input files are named `{first}-{last}.jpg` where first and last are the
  lowest and highest diagram numbers on the page (e.g., `463-468.jpg`)
- All six diagrams on a Book A page share the same side-to-move, indicated by a
  page header ("BLACK MOVES FIRST" / "WHITE MOVES FIRST")
- Book A answers scans are named `{first}-{last}.jpg` and placed in
  `data/book_a/answers/`

**Assumptions**
- The side-to-move header always appears on Book A diagram pages and is always
  readable by the Claude Vision API.
- The Claude Vision API can reliably extract FEN from clean chess diagram scans;
  the review stage exists to catch the cases where it cannot.
- The centipawn error threshold (100 cp, i.e., one pawn) is a reasonable default
  for flagging potentially erroneous book moves; the operator may adjust this in
  `config.yaml`.
- For Phase 1, the book's recommended move is not yet available; Stockfish analysis
  runs unconditionally to produce best-move and evaluation data, which will be
  compared against book moves once Phase 2 is complete.
- Book B board detection (Phase 3) is out of scope for this architecture document.

---

## 4. Architecture

### 4.1 Component Overview

Stages run sequentially. Stage 1b is Phase 2 only.

```
  raw scans          Stage 1 · extract.py        diagram images
  data/book_a/  ───► (rotate, slice grid)  ────► data/diagrams/
                             │
                             ▼
                     pipeline.db · diagrams table
                     (status = 'extracted')
                             │
                      [Phase 2 only]
  answers scans      Stage 1b · answers.py
  data/book_a/  ───► (Claude Vision OCR)   ────► pipeline.db · answers table
  answers/
                             │
                             ▼
                      Stage 2 · fen.py
                      (Claude Vision → FEN)
                             │
                             ▼
                     pipeline.db · diagrams table
                     (status = fen_ready | fen_review | fen_failed)
                             │
                    ┌────────┴────────┐
              fen_review?         fen_ready
                    │                 │
                    ▼                 │
             Stage 3 · review.py      │
             (Flask, human review)    │
                    │                 │
                    ▼                 │
             review_approved ─────────┘
                    │
                    ▼
             Stage 4 · analyze.py
             (Stockfish UCI + optional move comparison)
                    │
                    ▼
             pipeline.db · analysis table
                    │
                    ▼
             Stage 5 · report.py
             (Jinja2 HTML render)
                    │
                    ▼
             data/report.html
```

All stages share `pipeline.db` via `db.py`. No stage reads another stage's output files directly.

### 4.2 Data Flow

1. **Stage 1** (`extract.py`) reads raw page scans from `data/book_a/` or
   `data/book_b/`, crops individual diagram images to `data/diagrams/`, and
   inserts one row per diagram into the `diagrams` table with status `extracted`.

2. **Stage 1b** (`answers.py`, Phase 2 only) reads the answers scans from
   `data/book_a/answers/`, sends them to Claude Vision, and inserts one row per
   diagram number into the `answers` table with the move in descriptive notation.

3. **Stage 2** (`fen.py`) reads `diagrams` rows with status `extracted`, calls
   Claude Vision for each, and updates each row with the returned FEN,
   confidence score, and validity flag. Status advances to `fen_ready` (valid,
   high confidence), `fen_review` (low confidence or invalid), or `fen_failed`
   (Claude returned no usable FEN).

4. **Stage 3** (`review.py`) presents all `fen_review` rows to a human via a
   local Flask UI. The human approves, edits, or rejects each FEN. Status
   advances to `review_approved` or `review_rejected`.

5. **Stage 4** (`analyze.py`) reads all rows with status `fen_ready` or
   `review_approved`, runs Stockfish analysis, and inserts results into the
   `analysis` table. If an `answers` row exists for the diagram, the book move
   is included in the analysis.

6. **Stage 5** (`report.py`) reads `analysis` rows, filters for flagged
   positions, and renders `data/report.html` via a Jinja2 template.

### 4.3 Stage Execution

Each stage is a standalone script invoked from the command line. Stages are
idempotent: re-running a stage skips records already in the target status. A
`pipeline.py` driver script can run all stages in sequence.

---

## 5. Data Model

All state is stored in `data/pipeline.db` (SQLite).

### Table: `diagrams`

| Column          | Type    | Constraints             | Description                                        |
|-----------------|---------|-------------------------|----------------------------------------------------|
| `id`            | INTEGER | PRIMARY KEY             | Auto-increment                                     |
| `book_id`       | TEXT    | NOT NULL                | `'book_a'` or `'book_b'`                           |
| `diagram_number`| INTEGER | NOT NULL UNIQUE         | Canonical diagram number (e.g., 463)               |
| `page_file`     | TEXT    | NOT NULL                | Source scan filename (e.g., `463-468.jpg`)         |
| `image_path`    | TEXT    | NOT NULL UNIQUE         | Cropped diagram path (e.g., `data/diagrams/463.jpg`)|
| `side_to_move`  | TEXT    | NOT NULL                | `'w'` or `'b'` (from page header)                 |
| `fen`           | TEXT    |                         | FEN string (set by Stage 2)                        |
| `fen_confidence`| REAL    |                         | 0.0–1.0 self-reported by Claude                    |
| `fen_valid`     | INTEGER |                         | 1 if python-chess accepted the FEN, else 0         |
| `status`        | TEXT    | NOT NULL                | See status values below                            |
| `review_note`   | TEXT    |                         | Human comment from Stage 3                         |

**Status values (ordered):**
`extracted` → `fen_ready` | `fen_review` | `fen_failed` → `review_approved` | `review_rejected`

The `side_to_move` column is set at extraction time (Stage 1). The FEN stored
in this table always has the correct side-to-move field from this column.

### Table: `answers`

| Column          | Type    | Constraints        | Description                                        |
|-----------------|---------|--------------------|----------------------------------------------------|
| `id`            | INTEGER | PRIMARY KEY        |                                                    |
| `book_id`       | TEXT    | NOT NULL           |                                                    |
| `diagram_number`| INTEGER | NOT NULL UNIQUE    |                                                    |
| `descriptive`   | TEXT    |                    | Move in descriptive notation (e.g., `B-R3`); NULL if Claude could not extract it |
| `uci_move`      | TEXT    |                    | Resolved UCI move (e.g., `f1h3`), set by Stage 4  |
| `source`        | TEXT    | NOT NULL           | `'ocr'` or `'manual'`                             |

### Table: `analysis`

| Column              | Type    | Constraints        | Description                                        |
|---------------------|---------|--------------------|----------------------------------------------------|
| `id`                | INTEGER | PRIMARY KEY        |                                                    |
| `diagram_id`        | INTEGER | NOT NULL UNIQUE    | FK → `diagrams.id`                                 |
| `engine_best_move`  | TEXT    | NOT NULL           | Best move in UCI format                            |
| `engine_eval_cp`    | INTEGER | NOT NULL           | Centipawn eval of best move from the side-to-move's perspective (always ≥ book move eval) |
| `book_move_uci`     | TEXT    |                    | Book move in UCI (NULL if not yet available)       |
| `book_move_eval_cp` | INTEGER |                    | Eval after book move from the side-to-move's perspective (NULL if book move unavailable) |
| `centipawn_loss`    | INTEGER |                    | `engine_eval_cp - book_move_eval_cp` (always ≥ 0; NULL if N/A) |
| `flagged`           | INTEGER | NOT NULL DEFAULT 0 | 1 if `centipawn_loss >= threshold`                 |
| `depth`             | INTEGER | NOT NULL           | Stockfish search depth used                        |

**Centipawn perspective note:** All evaluations are stored from the **side-to-move's perspective**, not
White's. Use `score.pov(board.turn).score(mate_score=10000)` (python-chess) so that a higher number
always means a better position for the moving side regardless of colour. This ensures
`centipawn_loss = engine_eval_cp - book_move_eval_cp` is always ≥ 0 (the engine's best move is
never worse than the book move) and the threshold comparison is correct for both White and Black
positions.

---

## 6. Detailed Design

### 6.0 Database Module (`db.py`)

**Purpose:** Own the SQLite schema definition and provide a small set of shared
helpers used by all stage scripts. No stage script contains raw `CREATE TABLE`
SQL or opens the database path directly — they all go through `db.py`.

**Interface:**

```python
def get_connection(config: dict) -> sqlite3.Connection:
    """
    Open (or create) the SQLite database at config['data_dir']/pipeline.db.
    Enables WAL journal mode and foreign key enforcement.
    Returns an open connection with row_factory = sqlite3.Row.
    """

def init(config: dict) -> None:
    """
    Create all tables if they do not already exist (idempotent).
    Safe to call at the start of every stage script.
    Uses CREATE TABLE IF NOT EXISTS throughout.
    """
```

**Schema created by `init()`:**

```sql
CREATE TABLE IF NOT EXISTS diagrams (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id        TEXT    NOT NULL,
    diagram_number INTEGER NOT NULL UNIQUE,
    page_file      TEXT    NOT NULL,
    image_path     TEXT    NOT NULL UNIQUE,
    side_to_move   TEXT    NOT NULL CHECK(side_to_move IN ('w', 'b')),
    fen            TEXT,
    fen_confidence REAL,
    fen_valid      INTEGER,
    status         TEXT    NOT NULL,
    review_note    TEXT
);

CREATE TABLE IF NOT EXISTS answers (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id        TEXT    NOT NULL,
    diagram_number INTEGER NOT NULL UNIQUE,
    descriptive    TEXT,
    uci_move       TEXT,
    source         TEXT    NOT NULL CHECK(source IN ('ocr', 'manual'))
);

CREATE TABLE IF NOT EXISTS analysis (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    diagram_id        INTEGER NOT NULL UNIQUE REFERENCES diagrams(id),
    engine_best_move  TEXT    NOT NULL,
    engine_eval_cp    INTEGER NOT NULL,
    book_move_uci     TEXT,
    book_move_eval_cp INTEGER,
    centipawn_loss    INTEGER,
    flagged           INTEGER NOT NULL DEFAULT 0,
    depth             INTEGER NOT NULL
);
```

**Usage pattern in every stage script:**

```python
import db, yaml
config = yaml.safe_load(open('config.yaml'))
db.init(config)
conn = db.get_connection(config)
```

**Error handling:**
- If the database file cannot be created (e.g., `data/` directory does not
  exist), `get_connection()` raises `sqlite3.OperationalError`. Stage scripts
  do not catch this; they let it propagate and abort with a traceback.
- `init()` is safe to call multiple times; `CREATE TABLE IF NOT EXISTS`
  guarantees no-op on subsequent calls.

**Dependencies:** Python `sqlite3` (stdlib), pyyaml.

---

### 6.1 Stage 1 — Diagram Extraction (`extract.py`)

**Purpose:** Crop individual diagram images from full-page scans and record
them in the database.

**Satisfies:** Book A fixed-grid extraction (Phase 1 core).

**Interface:**
```
python extract.py [--book book_a|book_b] [--dry-run]
```
Reads page scans from `data/{book_id}/`. Writes cropped images to
`data/diagrams/`. Inserts rows into `diagrams`. Skips pages whose diagrams are
already in the database (idempotent by `diagram_number`).

**Behavior — Book A (grid mode):**

1. Scan `data/book_a/` for files matching `{first}-{last}.jpg`.
2. Parse `first` and `last` from the filename; derive `N = first` (the lowest
   diagram number on the page).
3. Read the page image with Pillow. **Rotate it 90° clockwise** using
   `image.rotate(-90, expand=True)` to convert from the stored landscape
   orientation to the book's natural portrait orientation (text upright).
   All subsequent steps operate on the rotated image.
4. **Side-to-move detection:** Send the **rotated** page image to the Claude
   Vision API with the prompt:
   > "Does this chess diagram page say 'WHITE MOVES FIRST' or 'BLACK MOVES
   > FIRST'? Reply with only 'w' or 'b'."
   Cache the result in a local dict keyed by filename so each page is sent
   only once.
5. **Divide into a 3-row × 2-column grid** with configurable margins
   (pixels inset from each cell edge, specified in `config.yaml` as
   `grid_margin_px`). Cell (row, col) coordinates (0-indexed, portrait):
   ```
   (0,0)=top-left    (0,1)=top-right
   (1,0)=mid-left    (1,1)=mid-right
   (2,0)=bot-left    (2,1)=bot-right
   ```
6. **Map grid cells to diagram numbers** using standard reading order:
   ```
   (0,0) → N     (e.g., 463)
   (0,1) → N+1   (e.g., 464)
   (1,0) → N+2   (e.g., 465)
   (1,1) → N+3   (e.g., 466)
   (2,0) → N+4   (e.g., 467)
   (2,1) → N+5   (e.g., 468)
   ```
   General rule: `diagram_number = N + 2*row + col`.
7. Crop each cell and save to `data/diagrams/{diagram_number}.jpg`.
8. Insert a row into `diagrams` with status `extracted`, `side_to_move` from
   step 4, and all other metadata. If a row with that `diagram_number` already
   exists, skip (do not overwrite).

**Error handling:**
- If the filename does not match `{int}-{int}.jpg`, log a warning and skip the file.
- If Claude returns anything other than `'w'` or `'b'` for side-to-move, log a
  warning, set `side_to_move = 'w'` as a default, and mark the page for
  human attention in a separate `warnings` log file (`data/warnings.log`).
- If a crop would fall outside the image bounds (indicating a misconfigured
  margin), raise a `ValueError` with the filename and cell coordinates.

**Dependencies:** Pillow, anthropic SDK, SQLite (via Python `sqlite3`), pyyaml.

---

### 6.2 Stage 1b — Answers Extraction (`answers.py`, Phase 2)

**Purpose:** Extract the book's recommended move for each Book A diagram from
the answers scans.

**Satisfies:** Phase 2 move comparison.

**Interface:**
```
python answers.py [--dry-run]
```
Reads answer scans from `data/book_a/answers/`. Inserts rows into `answers`.
Skips diagram numbers already in the `answers` table.

**Behavior:**

1. Scan `data/book_a/answers/` for files matching `{first}-{last}.jpg`.
2. For each file, call the Claude Vision API with the prompt:
   > "This image is a solutions page from a chess book. For each diagram number
   > from {first} to {last}, extract the first move given in the solution. The
   > moves are in descriptive notation (e.g., B-R3, NxP, Q-K8 mate). Return a
   > JSON object mapping diagram number (as a string) to the first move (as a
   > string). If no move can be found for a diagram number, map it to null."
3. Parse the returned JSON. For each entry, insert a row into `answers` with
   `source = 'ocr'`. If Claude returns `null` for a diagram number, insert with
   `descriptive = NULL` and log a warning.
4. After all OCR is complete, a human can manually correct nulls or wrong values
   by directly editing the database (no separate UI is provided for Phase 1/2).

**Error handling:**
- If Claude returns malformed JSON, log the raw response to `data/warnings.log`
  and skip the file. The operator must re-run or manually insert records.
- If the filename does not match `{int}-{int}.jpg`, skip with a warning.

**Dependencies:** anthropic SDK, sqlite3, pyyaml.

---

### 6.3 Stage 2 — FEN Extraction (`fen.py`)

**Purpose:** Convert each cropped diagram image to a FEN string using Claude
Vision.

**Interface:**
```
python fen.py [--limit N] [--model claude-sonnet-4-6]
```
`--limit N` processes at most N images (useful for testing). Reads diagrams
with status `extracted`. Updates each row in `diagrams` with FEN data and new
status.

**Behavior:**

1. Query `diagrams` WHERE `status = 'extracted'`.
2. For each row, read the cropped image at `image_path`.
3. Call Claude Vision API with:
   > "This is a chess diagram. The side to move is {side_to_move spelled out:
   > 'White' or 'Black'}. Please return:
   > 1. The FEN string for this position (include the side-to-move field).
   > 2. Your confidence that the FEN is correct, as a decimal between 0 and 1.
   > Return JSON: {"fen": "...", "confidence": 0.95}"
4. Parse the JSON response.
5. Validate the FEN using `chess.Board(fen)` from `python-chess`; catch
   `chess.InvalidMoveError` and `ValueError`.
6. Determine status:
   - `fen_ready` if valid AND `confidence >= config.fen_confidence_threshold`
     (default 0.85)
   - `fen_review` if valid but confidence below threshold, OR if FEN is invalid
     but Claude returned something parseable
   - `fen_failed` if Claude returned no FEN or response was unparseable
7. Update the `diagrams` row with `fen`, `fen_confidence`, `fen_valid`, and
   `status`. Commit after each row (not at the end) so crashes do not lose work.

**Error handling:**
- If the Claude API call fails (network error, rate limit), wait 5 seconds and
  retry once. On second failure, set `status = 'fen_failed'` and log to
  `data/warnings.log`.
- If JSON parsing fails, set `status = 'fen_failed'`.
- Never call the API for a row that already has `status != 'extracted'`
  (idempotency guarantee).

**Dependencies:** anthropic SDK, python-chess, sqlite3, Pillow (to read images
as base64 for the API), pyyaml.

---

### 6.4 Stage 3 — Human Review (`review.py`)

**Purpose:** Allow a human to inspect and correct uncertain FEN strings.

**Interface:**
```
python review.py [--port 5000]
```
Starts a local Flask web server. The operator opens `http://localhost:5000` in
a browser. The server runs until all flagged positions have been reviewed, or
the operator stops it with Ctrl-C.

**Behavior:**

The Flask app serves a single-page review interface:

- **`GET /`**: Redirects to `/review/next`.
- **`GET /review/next`**: Queries for the first `diagrams` row WHERE
  `status = 'fen_review'`, ordered by `diagram_number`. If none, renders a
  "Review complete" page. Otherwise redirects to `/review/{id}`.
- **`GET /review/{id}`**: Renders a page showing:
  - The original cropped diagram image (`<img src="/image/{id}">`).
  - An SVG board rendered from the current FEN using
    `chess.svg.board(chess.Board(fen))` from python-chess.
  - The current FEN string in an editable `<textarea>`.
  - Three buttons: **Approve**, **Save & Approve** (after editing), **Reject**.
  - The diagram number, book, confidence score.
- **`GET /image/{id}`**: Returns the diagram image file from disk.
- **`POST /review/{id}`**: Accepts `action` (`approve`, `reject`) and optional
  `fen` field. Updates the `diagrams` row:
  - `approve`: sets `status = 'review_approved'`, updates `fen` if a new value
    was submitted, re-validates FEN.
  - `reject`: sets `status = 'review_rejected'`.
  Redirects to `/review/next`.

**Error handling:**
- If the FEN submitted in a POST is invalid (python-chess rejects it), return
  the form with an error message; do not save and do not advance status.
- If `{id}` does not exist or is not in `fen_review` status, return HTTP 404.

**Dependencies:** Flask, python-chess, sqlite3.

---

### 6.5 Stage 4 — Stockfish Analysis (`analyze.py`)

**Purpose:** Evaluate each validated position with Stockfish and, where a book
move is available, compare it against the engine's recommendation.

**Interface:**
```
python analyze.py [--depth 20]
```
Reads `diagrams` rows with `status IN ('fen_ready', 'review_approved')` that
do not yet have a corresponding `analysis` row. Writes to `analysis`.

**Behavior:**

1. Open Stockfish via `chess.engine.SimpleEngine.popen_uci(config.stockfish_path)`.
2. For each eligible `diagrams` row:
   a. Construct `chess.Board(fen)`.
   b. Call `engine.analyse(board, chess.engine.Limit(depth=config.analysis_depth))`.
      Record `engine_best_move` (UCI) and `engine_eval_cp` (centipawns, from
      the side-to-move's perspective; use `score.pov(board.turn).score(mate_score=10000)`).
   c. If an `answers` row exists for this `diagram_number` AND `uci_move` is not
      yet set: call Claude API with:
      > "Given this chess position in FEN: {fen}
      > The side to move is {side}. The book recommends the move '{descriptive}'
      > in descriptive notation. What is this move in UCI format (e.g., e2e4)?
      > Reply with only the UCI move string."
      Validate the returned string as a legal move in the position using
      `chess.Move.from_uci()` and `board.is_legal(move)`. Store in
      `answers.uci_move`.
   d. If `uci_move` is now available, push the book move onto a copy of the
      board and call `engine.analyse()` again. Retrieve `book_move_eval_cp`
      using `score.pov(board.turn).score(mate_score=10000)` **on the original
      board** (before the push), so the perspective is the same side as for
      `engine_eval_cp`. Compute `centipawn_loss = engine_eval_cp - book_move_eval_cp`.
      Set `flagged = 1` if `centipawn_loss >= config.centipawn_error_threshold`.
   e. Insert a row into `analysis`. Commit after each row.
3. Close the Stockfish engine.

**Error handling:**
- If Stockfish binary is not found, raise `FileNotFoundError` with the
  configured path and abort.
- If `engine.analyse()` raises an exception, log to `data/warnings.log`, set
  the `analysis` row's `flagged = 0`, and continue.
- If the Claude move-resolution call returns an illegal move, log the raw
  response and leave `answers.uci_move = NULL` (do not flag).
- Never insert a duplicate `analysis` row for a `diagram_id` (the UNIQUE
  constraint enforces this; catch `IntegrityError` and skip).

**Dependencies:** python-chess, anthropic SDK (Phase 2 only), sqlite3, pyyaml.

---

### 6.6 Stage 5 — Reporting (`report.py`)

**Purpose:** Generate a human-readable HTML report of flagged positions.

**Interface:**
```
python report.py [--output data/report.html]
```

**Behavior:**

1. Query:
   ```sql
   SELECT d.diagram_number, d.book_id, d.image_path, d.fen,
          a.engine_best_move, a.engine_eval_cp,
          a.book_move_uci, a.book_move_eval_cp, a.centipawn_loss
   FROM analysis a
   JOIN diagrams d ON a.diagram_id = d.id
   WHERE a.flagged = 1
   ORDER BY d.book_id, d.diagram_number
   ```
2. For each row, generate an inline SVG board using
   `chess.svg.board(chess.Board(fen))`.
3. Render `templates/report.html.j2` with Jinja2, passing the query results and
   SVG strings.
4. Write the rendered HTML to the output path.

**Template structure (`templates/report.html.j2`):**
- One section per book.
- Within each section: one card per flagged diagram showing the diagram image,
  the SVG board, the FEN, the engine best move and eval, the book move and eval
  (if available), and the centipawn loss.

**Error handling:**
- If no flagged positions exist, write a report stating "No errors found."
- If `image_path` for a diagram does not exist on disk, include a placeholder
  `[image not found]` rather than crashing.

**Dependencies:** python-chess, Jinja2, sqlite3.

---

### 6.7 Pipeline Driver (`pipeline.py`)

**Purpose:** Run all stages in sequence. Not required; stages can be run
individually. Provided for convenience.

**Interface:**
```
python pipeline.py [--book book_a|book_b] [--phase 1|2]
```

Calls each stage script as a subprocess in order:
`extract` → `answers` (Phase 2 only) → `fen` → `review` → `analyze` → `report`

**Stage 3 handoff:** `review.py` is an interactive web server, not a
batch job. `pipeline.py` handles it as follows:

1. Query the database: if zero rows have `status = 'fen_review'`, skip Stage 3
   entirely and proceed directly to `analyze`.
2. If review rows exist, print:
   ```
   [chesspipe] N positions need review. Run:  python review.py
   Then, when review is complete, re-run:    python pipeline.py --phase 1
   ```
   and exit with code 0. The operator completes review in their browser, then
   re-runs `pipeline.py`. On the second run, the review queue is empty and
   the pipeline continues from `analyze` onward.

This avoids any polling or subprocess lifetime management. Stages before the
review checkpoint (`extract`, `fen`) are idempotent and produce no duplicate
work on re-run.

---

## 7. Configuration (`config.yaml`)

```yaml
# Paths
stockfish_path: /usr/local/bin/stockfish
data_dir: ./data

# Claude
claude_model: claude-sonnet-4-6
# API key is read from the ANTHROPIC_API_KEY environment variable; do not store here.

# Stage 1 — Grid extraction (Book A)
# Scans are stored rotated 90° CW; extractor rotates them to portrait before slicing.
grid_rows: 3
grid_cols: 2
grid_margin_px: 10        # pixels to inset from each cell edge when cropping

# Stage 2 — FEN extraction
fen_confidence_threshold: 0.85   # below this → fen_review status

# Stage 4 — Stockfish analysis
analysis_depth: 20
centipawn_error_threshold: 100   # flag if book move loses >= 1 pawn vs best
```

---

## 7b. Dependencies (`requirements.txt`)

```
anthropic>=0.25.0
chess>=1.10.0
Flask>=3.0.0
Jinja2>=3.1.0
opencv-python-headless>=4.9.0
Pillow>=10.0.0
PyYAML>=6.0.1
```

Notes:
- `opencv-python-headless` is used for Phase 3 (Book B board detection). It is
  listed now so the environment is consistent from the start; Stage 1 (grid
  mode) does not import it.
- `Jinja2` is a Flask dependency and will be present automatically, but is
  listed explicitly because `report.py` uses it directly.
- `stockfish` (PyPI wrapper) is intentionally omitted; `python-chess` provides
  its own UCI engine interface (`chess.engine.SimpleEngine`).
- Pin to exact versions before shipping using `pip freeze > requirements.txt`
  after a successful end-to-end test run.

---

## 8. File and Directory Plan

```
chesspipe/
├── config.yaml                  MODIFY  add grid settings, threshold fields
├── extract.py                   CREATE  Stage 1 — diagram extraction
├── answers.py                   CREATE  Stage 1b — answers OCR (Phase 2)
├── fen.py                       CREATE  Stage 2 — FEN extraction via Claude
├── review.py                    CREATE  Stage 3 — Flask human review UI
├── analyze.py                   CREATE  Stage 4 — Stockfish analysis
├── report.py                    CREATE  Stage 5 — HTML report generation
├── pipeline.py                  CREATE  convenience driver script
├── db.py                        CREATE  database schema creation and shared helpers
├── requirements.txt             CREATE  pinned dependencies
├── templates/
│   └── report.html.j2           CREATE  Jinja2 report template
└── data/
    ├── book_a/                  EXISTS  Book A page scans
    │   └── answers/             EXISTS  Book A answers scans
    ├── book_b/                  EXISTS  Book B page scans
    ├── diagrams/                CREATE  cropped diagram images (output of Stage 1)
    ├── pipeline.db              CREATE  SQLite state store (all stages)
    ├── report.html              CREATE  final error report (output of Stage 5)
    └── warnings.log             CREATE  append-only log of non-fatal errors
```

---

## 9. Key Design Decisions

| Decision | Alternatives Considered | Choice | Rationale |
|----------|------------------------|--------|-----------|
| State store | Two flat JSON files (README), one JSON file, SQLite | SQLite (`pipeline.db`) | Atomic row-level commits; idempotent stage runs; easy querying for the report; no custom serialization code |
| Review UI | Tkinter GUI, curses terminal, Flask web app | Flask (`localhost:5000`) | No native GUI dependencies; works over SSH; browser renders SVG natively; easiest to develop and debug |
| Side-to-move detection | Hardcode per book, ask human, OCR from page header via Claude | Claude Vision API on full page image | The header text is present and readable; one API call per page (not per diagram); cost is negligible vs. diagram FEN calls |
| Answers extraction | Manual transcription, local OCR (Tesseract), Claude Vision | Claude Vision API | The answers text is small, dense, multi-column, and in an archaic notation; Claude Vision outperforms Tesseract on this kind of text |
| Scan rotation | Process landscape image directly with transposed grid math, rotate before slicing | Rotate 90° CW to portrait before slicing | Keeps all downstream grid logic simple and in the natural portrait coordinate space; Pillow rotation is trivial |
| Grid mapping | Sequential left-to-right top-to-bottom (N, N+1, N+2, …), column-major | Left column = odd numbers (N, N+2, N+4), right column = even (N+1, N+3, N+5) | Required by the actual book layout; the formula `N + 2*row + col` is simple and directly encodes this pattern |
| Idempotency | Re-process everything, external lock files, status field | Status field in `diagrams` table | Simple, inspectable, no external dependencies; the operator can reset individual rows to re-process them |
| Phase 2 as a separate script | Integrate into Stage 1, integrate into Stage 4 | Separate `answers.py` | Keeps each script focused; answers extraction can be run independently; failure in answers does not block Phase 1 analysis |

---

## 10. Requirement Traceability

| Feature | Design Section | Files |
|---------|---------------|-------|
| Extract Book A diagrams (fixed grid) | §6.1 | `extract.py`, `db.py` |
| Rotate scans to portrait before slicing | §2.1, §6.1 | `extract.py` |
| Grid: 3 rows × 2 cols, left=odd/right=even numbering | §2.1, §6.1 | `extract.py` |
| Side-to-move from page header | §2.2, §6.1 | `extract.py` |
| Detect Book B diagrams (flow layout) | Phase 3, out of scope | — |
| FEN extraction via Claude Vision | §6.3 | `fen.py` |
| FEN validation | §6.3 | `fen.py` |
| Human review of uncertain FENs | §6.4 | `review.py`, `templates/report.html.j2` |
| Stockfish analysis | §6.5 | `analyze.py` |
| Book move extraction from answers scans | §6.2 | `answers.py` |
| Descriptive → UCI move resolution | §6.5 (step 2c) | `analyze.py` |
| Error flagging by centipawn threshold | §6.5, §5 | `analyze.py`, `config.yaml` |
| HTML report of flagged positions | §6.6 | `report.py`, `templates/report.html.j2` |
| Idempotent stage execution | §5, §6.0–6.5 | `db.py`, all stage scripts |
| Pipeline state persistence | §5, §6.0 | `db.py`, `data/pipeline.db` |
| Database schema and shared helpers | §6.0 | `db.py` |
| Stage 3 / pipeline handoff protocol | §6.7 | `pipeline.py` |
| Centipawn loss from side-to-move perspective | §5, §6.5 | `analyze.py` |
| Configuration file | §7 | `config.yaml` |
| Python dependencies | §7b | `requirements.txt` |

---

## 11. Testing Strategy

**Unit tests** (use `pytest`):

- `test_extract.py`:
  - Given a synthetic landscape image containing a 3×2 grid of colored cells
    (after the mandatory 90° CW rotation), verify that `extract_grid_cells()`
    returns the correct (diagram_number, cropped_image) pairs using the formula
    `N + 2*row + col`.
  - Verify that rotating a landscape image 90° CW produces a taller-than-wide
    (portrait) image before slicing.
  - Verify that a filename `463-468.jpg` is parsed to `first=463, last=468`.
  - Verify that a filename not matching `{int}-{int}.jpg` is skipped without
    raising an exception.

- `test_fen.py`:
  - Given a mock Claude response returning valid JSON with FEN and confidence,
    verify the correct status is assigned (`fen_ready` vs `fen_review`).
  - Given a mock Claude response returning invalid FEN, verify `status = 'fen_failed'`.
  - Verify that a diagram already at status `fen_ready` is skipped (not re-called).

- `test_analyze.py`:
  - Given a known FEN with **Black to move** and a suboptimal book move, verify
    that `centipawn_loss` is positive (not negative) — confirming the
    side-to-move perspective is used, not White's perspective.
  - Given `centipawn_loss >= 100`, verify `flagged = 1`.
  - Given `centipawn_loss < 100`, verify `flagged = 0`.
  - Given no `answers` row, verify analysis completes with `book_move_uci = NULL`
    and `flagged = 0`.

- `test_db.py`:
  - Verify schema creation is idempotent (running `db.init()` twice does not
    raise an error).
  - Verify UNIQUE constraints on `diagrams.diagram_number` and
    `analysis.diagram_id`.

**Integration tests:**

- Run Stage 1 against the real `data/book_a/463-468.jpg` scan. Verify that
  six diagram files are created in `data/diagrams/` and six rows appear in
  `diagrams` with correct `diagram_number` values (463–468) and
  `side_to_move = 'b'` (since the page says "BLACK MOVES FIRST").

- Run Stage 2 against one real diagram image. Verify that a FEN is returned
  and accepted by python-chess.

**Manual verification:**

- After Stage 3 (review), visually compare the SVG-rendered board against the
  original scan for a sample of approved positions.
- After Stage 5 (report), open `report.html` and verify flagged diagrams look
  plausible (wrong-side tactic, etc.) rather than being extraction artifacts.

---

## 12. Open Questions

1. **Book A grid margin values:** The exact pixel insets needed to cleanly crop
   each diagram (excluding the diagram number caption and any page border) must
   be determined empirically from the actual scans. The default `grid_margin_px: 10`
   is a placeholder.

2. **Consistency of the grid numbering pattern:** The mapping (left column =
   odd numbers N, N+2, N+4; right column = even numbers N+1, N+3, N+5) is
   observed in `463-468.jpg`. Confirm this holds for all Book A pages. If any
   page uses a different numbering scheme (e.g., sequential left-to-right), a
   per-page override in `config.yaml` will be needed.

3. **Side-to-move consistency within a page:** This architecture assumes all six
   diagrams on a Book A page share the same side-to-move. Confirm this is true
   for all pages. If some pages mix sides, the per-diagram side-to-move will
   need to come from a different source (e.g., manual annotation or caption OCR).

4. **Centipawn threshold calibration:** The 100 cp threshold is a reasonable
   starting point but may produce too many or too few flags in practice. Plan
   for an initial calibration pass after Phase 1 is complete.

5. **Book B layout:** Phase 3 (flow-layout board detection for Book B) is
   explicitly deferred. When it is designed, the `extract.py` detect-mode
   branch and the `book_b` input path will need to be defined.
