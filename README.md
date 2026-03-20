# chesspipe
Chess image processing pipeline


## Overview

This project builds an automated pipeline to extract chess diagrams from scanned book pages,
convert them to FEN notation, and analyze the resulting positions with Stockfish. The primary
goal is to detect errors in old chess books — positions where the book's recommended move
disagrees with modern engine analysis.

---

## Motivation

Old chess books contain many errors: misprinted diagrams, faulty analysis, and moves that were
considered sound in their era but are refuted by modern engines. This pipeline automates the
tedious work of checking every diagram in a book, flagging candidates for human review.

---

## Input Sources

**Book A — Fixed Layout**
- Approximately 150 scanned pages
- Six diagrams per page in a consistent 3-row × 2-column grid
- Answers given in descriptive (English) algebraic notation (e.g., P-K4, N-KB3)
- Diagram extraction uses fixed region slicing (no detection needed)

**Book B and Others — Flow Layout**
- Diagrams embedded in text columns at variable positions
- Fewer diagrams per book (dozens rather than hundreds)
- Diagram extraction requires chess board pattern detection

All input images are provided as individual PNG or JPG files (not PDFs).

---

## Pipeline Stages

### Stage 1 — Diagram Extraction (`extract.py`)

Crops individual chess diagrams from full page scans.

- **Grid mode** (Book A): Divide each page into a fixed 3×2 grid and slice each cell.
  Includes configurable margins to exclude page numbers and captions.
- **Detect mode** (Book B): Use OpenCV to locate chess board patterns based on grid
  line structure and alternating square colors. Outputs bounding boxes, then crops.
- Output: Individual diagram images saved to `data/diagrams/`, named by
  `{book}_{page:04d}_{index}.png`

### Stage 2 — FEN Extraction (`fen.py`)

Converts each cropped diagram image to a FEN string using the Claude Vision API
(model: `claude-opus-4-6` or `claude-sonnet-4-6`).

- Sends each diagram image to Claude with a prompt requesting the FEN string
- Receives FEN plus a self-reported confidence level
- Validates the FEN using `python-chess` (catches illegal positions)
- Flags low-confidence or invalid FENs for human review
- Output: `data/positions.json` — list of records with image path, FEN, confidence,
  validity status

### Stage 3 — Human Review (`review.py`)

Allows a human to inspect and correct FEN strings before analysis.

- Displays the cropped diagram image alongside a rendered board (SVG via `python-chess`)
  so the human can compare the scan with the extracted FEN
- Presents any flagged positions (low confidence, invalid) for correction
- Allows editing the FEN string interactively
- Saves corrections back to `data/positions.json`
- Approved positions are marked as ready for analysis

### Stage 4 — Stockfish Analysis (`analyze.py`)

Evaluates each validated position using Stockfish via the UCI protocol.

- Uses `python-chess`'s `chess.engine.SimpleEngine.popen_uci()` to communicate with
  the Stockfish executable (path specified in config)
- For each position: computes the best move and centipawn evaluation at a configurable
  search depth (default: depth 20)
- If the book's given move is available (Phase 2), plays that move and compares the
  resulting evaluation against the best line
- Flags positions where the book move loses significant material or eval relative to
  the engine's recommendation
- Output: `data/analysis.json` — FEN, best move, eval, book move (if known), error flag

### Stage 5 — Reporting (`report.py`)

Generates a human-readable report of flagged positions.

- Lists all positions flagged as potential errors, grouped by book and page
- For each flagged position: shows the diagram image path, FEN, engine best move,
  book move (if known), and centipawn loss
- Output: `data/report.html` — rendered with board diagrams for easy review

---

## Phase 2 — Descriptive Notation Parsing

Book A provides answers in old descriptive notation (e.g., `P-K4`, `N-KB3`, `QxP`).
In Phase 2, the pipeline will:

1. Extract the given move text (via OCR or manual transcription)
2. Use the Claude API — given the FEN and descriptive move string — to resolve the
   move to UCI format (e.g., `e2e4`)
3. Feed both the position and the book move into Stage 4 for automated comparison
4. This enables fully automated error detection for Book A without manual move lookup

---

## Data Layout

```
chess-pipeline/
├── config.yaml               # Stockfish path, API key ref, depth, thresholds
├── extract.py
├── fen.py
├── review.py
├── analyze.py
├── report.py
├── requirements.txt
└── data/
    ├── pages/
    │   ├── book_a/           # input scanned pages for Book A
    │   └── book_b/           # input scanned pages for Book B
    ├── diagrams/             # cropped diagram images (output of Stage 1)
    ├── positions.json        # FENs + metadata + review status (Stage 2/3)
    ├── analysis.json         # Stockfish results (Stage 4)
    └── report.html           # final error report (Stage 5)
```

---

## Dependencies

| Package | Purpose |
|---|---|
| `opencv-python` | Image processing, board detection |
| `Pillow` | Image cropping and saving |
| `anthropic` | Claude Vision API (FEN extraction, notation parsing) |
| `python-chess` | FEN validation, board rendering (SVG), UCI engine interface |
| `stockfish` | (optional wrapper — `python-chess` engine API used directly) |
| `pyyaml` | Configuration file parsing |
| `jinja2` | HTML report templating |

Python 3.11+. Stockfish built from source, path specified in `config.yaml`.

---

## Configuration (`config.yaml`)

```yaml
stockfish_path: /path/to/stockfish   # to be specified
analysis_depth: 20
centipawn_error_threshold: 100       # flag if book move loses >= 1 pawn vs best
claude_model: claude-sonnet-4-6
data_dir: ./data
```

---

## Development Phases

| Phase | Scope |
|---|---|
| 1 | Extract + FEN + Review + Stockfish eval (no move comparison) |
| 2 | Descriptive notation parsing, automated move verification for Book A |
| 3 | Flow-layout board detection for Book B and other books |

