# Session Log — 2026-04-13 (Session 2)
**Project:** ANPR Accuracy Analysis — Genting JPO  
**Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`  
**Duration context:** ~1.5 hours (approx. 11:37 – 13:00 MYT, same day as Session 1)

---

## Summary

This session extended the ANPR analysis pipeline (v1 from earlier today) with three major upgrades: (1) switching from the Excel accuracy flag to a new CSV that includes a `human_ground_truth` column annotated by human reviewers, giving a proper gold-standard baseline; (2) upgrading the VLM ground truth from Gemini 2.0 Flash to Gemini 2.5 Pro (the best available paid model) to maximise vision accuracy; and (3) adding full PDF report generation — two separate PDFs showing annotated plate images with YOLO bounding boxes + per-character confidence (matching a client-provided visual style), and a Gemini prediction card for each plate. By end of session all outputs were generated: a CSV, a 4-sheet Excel workbook, a YOLO annotation PDF, and a Gemini Vision PDF, with the headline result that Gemini 2.5 Pro achieved 80% exact match against human ground truth vs the deployed system's 2.5%.

---

## What Was Built

### `anpr_analysis_v2.py` — Upgraded Pipeline
- Complete rewrite of the v1 pipeline (~480 lines) with human GT support, Gemini 2.5 Pro, and PDF generation.
- **Location:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/anpr_analysis_v2.py`
- **Data source changed:** from Excel (`ACCURACY ?` column with 0/1) to CSV with explicit `human_ground_truth` column
- **Key new capabilities:**
  - Parses `human_ground_truth` field; marks 23/63 rows as "not clear" (human couldn't read the plate)
  - Calls `gemini-2.5-pro` model via `google-genai` SDK
  - Draws YOLO bounding boxes on each plate image with colour-coded boxes per character and confidence labels (matching the dark-theme card style shown by the client)
  - Draws a Gemini prediction card per plate (plate image + Gemini text + Human GT + System plate)
  - Builds two full PDFs via ReportLab: one for YOLO annotations, one for Gemini predictions
  - Outputs: CSV, 4-sheet Excel (Summary, Comparison, YOLO_CharDetail, GT_Readable_Only), two PDFs

### YOLO Annotation Cards (`annotated_yolo/yolo_row*.png`)
- Per-plate dark-theme card image: left side = plate crop with coloured YOLO bboxes + `CHAR (XX.X%)` label above each box; right side = dark panel listing each detected character with its colour swatch and confidence %.
- Style matched to the client's example screenshot (dark background, per-character colour coding, confidence as `XX.X%`).
- **Location:** `report/analysis_v2/annotated_yolo/yolo_row0000.png` … `yolo_row0062.png`

### Gemini Prediction Cards (`annotated_yolo/gemini_row*.png`)
- Per-plate card: plate image on left; right panel shows Gemini 2.5 Pro prediction (green), System plate (grey), Human GT (amber).
- **Location:** `report/analysis_v2/annotated_yolo/gemini_row0000.png` … `gemini_row0062.png`

### PDF Reports
- `yolo_model_report_20260413_114048.pdf` — 63-plate YOLO annotation PDF with comparison table per plate (System / YOLO / Gemini / Human GT, with ✓/✗ match indicators).
- `gemini_report_20260413_114048.pdf` — 63-plate Gemini Vision PDF, same structure.
- **Location:** `report/analysis_v2/`

### CSV & Excel Outputs
- `report_20260413_114048.csv` — full flat comparison table, one row per plate.
- `report_20260413_114048.xlsx` — 4 sheets: Summary metrics, full Comparison, YOLO_CharDetail (one row per detected character), GT_Readable_Only (40 rows where human GT was available).
- **Location:** `report/analysis_v2/`

---

## Key Decisions & Rationale

| Decision | Choice Made | Why |
|---|---|---|
| Ground truth source | `human_ground_truth` column from new CSV | Human annotations are the true gold standard; previous EasyOCR/Gemini-only GT was a proxy |
| VLM upgrade | `gemini-2.5-pro` (from `gemini-2.0-flash`) | Paid API now intended; 2.5 Pro is the most capable Gemini vision model available (confirmed via `client.models.list()`) |
| 23 "not clear" rows | Excluded from accuracy metrics, retained in output | Human reviewers marked these as unreadable — computing accuracy against them would be meaningless noise |
| PDF generation library | ReportLab (`reportlab`) | Mature, widely used, supports mixed image + table layout natively; alternative `fpdf2` was installed but not needed |
| YOLO card style | Dark background (#1e1e1e), colour-coded bboxes, `CHAR (XX.X%)` labels | Client provided exact visual reference screenshot — matched as closely as possible with OpenCV |
| Separate PDFs for YOLO and Gemini | Two PDFs | Client wanted to see YOLO detection visuals separately from VLM predictions for direct comparison |
| Accuracy metrics | Only on `human_gt_readable == True` rows (40/63) | Prevents the 23 "not clear" plates from dragging down all metrics unfairly |

---

## Findings & Observations

1. **Human GT coverage: 40/63 readable** — 23 plates were marked "not clear" by human reviewers. These are predominantly WRONG LANE DETECTION (8), GLITCHY CAMERA (9), and a few BLOCKED/BROKEN cases. This is expected and represents a hardware/placement failure category, not a model failure.

2. **Gemini 2.5 Pro: 80.0% exact match, 92.8% avg char accuracy** against human GT on 40 readable rows. This is a strong result confirming Gemini 2.5 Pro is a reliable VLM ground truth for Malaysian plates.

3. **YOLO best.pt: 35.0% exact match, 64.3% avg char accuracy** against human GT. A 14× improvement over the deployed system (2.5%), confirming the model weights are sound and the production preprocessing pipeline is the bottleneck.

4. **Deployed system: 2.5% exact match, 51.2% avg char accuracy** — only 1 out of 40 readable plates was read correctly. The system routinely adds/drops characters (e.g. `UITM447` instead of `UTM447`) or confuses similar-looking characters.

5. **EasyOCR: 7.5% exact match, 35.3% char accuracy** — worst performer. Not suitable for Malaysian plates without domain fine-tuning. Inconsistent on blurry/angled crops.

6. **Gemini 2.5 Pro correctly handles special plates** — reads `PUTRAJAYA8220` correctly, recognises Putrajaya-format plates where the system fails. Also correctly reconstructs partial plates from glitchy images more often than YOLO.

7. **YOLO systematic `I` → drop pattern** — UITM plates consistently read as `UTM...` (e.g., UITM447→UTM447, UITM7981→UTM7981, UITM5535→UTM5535). Class 18 (`I`) is likely underrepresented in training data, causing it to be suppressed at NMS.

8. **YOLO double-detection issue on long plates** — some plates get extra characters (e.g., `UTEM1212` predicted as `UTEM121112` — the `1` is double-detected). This is an NMS IOU threshold issue; lowering from 0.45 may help.

9. **Average YOLO confidence: 0.751** (down from 0.799 in v1 — same model, but the v2 run includes more partial/blurry plates that weren't excluded). Individual character confidence ranged from ~0.25 to 0.99.

10. **`google-generativeai` package is deprecated** — must use `google-genai` (the new SDK). The old package still installs but prints a `FutureWarning` and will stop receiving updates.

11. **Gemini occasional misreadings on very dark/blurry images** — on glitchy camera cases, Gemini sometimes hallucinates plausible-looking but incorrect plates (e.g. reads `JWG3108` for a `JWU300` wrong-lane image). These are cases where the human also marked "not clear", so they are excluded from metrics.

---

## Problems & Fixes

| Problem | Root Cause | Fix Applied |
|---|---|---|
| Anthropic API `BadRequestError: credit balance too low` | Anthropic account had no credits despite valid key | Switched to Google Gemini API entirely; Anthropic key abandoned for this session |
| `google.generativeai` deprecated warning | Old SDK package installed; new `google-genai` package needed | Installed `google-genai`; rewrote Gemini calls to use `google.genai.Client` and `types.Part.from_bytes()` |
| HuggingFace free serverless VLM support very limited | HF transitioned to "Inference Providers" model; most VLMs require paid partner tiers | Chose Google Gemini free tier (then upgraded to paid Gemini 2.5 Pro) instead of HF |
| `gemini-2.0-flash` was too generic | User requested "better VLM since it's paid API" | Switched to `gemini-2.5-pro` after confirming it appeared in `client.models.list()` |
| YOLO bbox colours needed to match client's dark-theme screenshot | Client provided reference screenshot with purple/teal coloured bboxes per character | Implemented `BBOX_COLOURS` palette (10 BGR colours), cycled per-detection index; matched dark `#1e1e1e` background |
| ReportLab image sizing broke on very wide annotation cards | Card width exceeded A4 page width | Added `disp_w = min(PAGE_W, cw)` clamp with aspect-ratio-preserving height |

---

## ML / Training Config

### YOLOv5 Inference (unchanged from v1)
```python
model.conf = 0.25   # confidence threshold
model.iou  = 0.45   # NMS IOU — consider lowering to 0.35 to fix double-detections
```

### Recommended next training adjustments
- Increase `I` (class 18) sample count in training data — currently suppressed on UITM plates
- Add Putrajaya / UITM / UTEM plate examples to training set (special formats currently mishandled)
- Consider lowering NMS IOU to 0.35–0.40 to reduce double-detection on wide plates
- Run inference at higher resolution input (640→1280) for small character plates

---

## Files Created / Modified

| File | Location | Purpose |
|---|---|---|
| `anpr_analysis_v2.py` | `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/` | Full v2 pipeline — human GT, Gemini 2.5 Pro, YOLO, PDF generation |
| `report_20260413_114048.csv` | `report/analysis_v2/` | Flat comparison table, all 63 rows |
| `report_20260413_114048.xlsx` | `report/analysis_v2/` | 4-sheet Excel workbook |
| `yolo_model_report_20260413_114048.pdf` | `report/analysis_v2/` | YOLO annotation PDF (63 plates, bbox overlays, confidence tables) |
| `gemini_report_20260413_114048.pdf` | `report/analysis_v2/` | Gemini 2.5 Pro prediction PDF (63 plates) |
| `annotated_yolo/yolo_row*.png` (63 files) | `report/analysis_v2/annotated_yolo/` | Per-plate YOLO dark-theme annotation card images |
| `annotated_yolo/gemini_row*.png` (63 files) | `report/analysis_v2/annotated_yolo/` | Per-plate Gemini prediction card images |

---

## Next Steps / Open Questions

- [ ] Investigate production preprocessing pipeline — what transforms are applied before feeding frames to the deployed YOLO model? Compare against the raw plate crops used here.
- [ ] Fix YOLO `I` suppression — inspect training data distribution for class 18 (`I`) vs class 1 (`1`) on UITM plates; augment training set.
- [ ] Lower NMS IOU to 0.35–0.40 to reduce double-detections on wide plates (`UTEM121112` → `UTEM1212`).
- [ ] Add per-remark-category accuracy breakdown in the PDF (section header per fault type: MISDETECTION vs GLITCHY CAMERA vs WRONG LANE).
- [ ] Package pipeline as CLI with `argparse` — client should be able to run `python anpr_analysis_v2.py --csv <path> --api-key <key>` without editing source.
- [ ] Retrain YOLO with additional special-format plates (Putrajaya, UITM, UTEM, JPJ non-standard).
- [ ] Add bounding box confidence histogram chart to the Excel Summary sheet.
- [ ] Fund Anthropic API and add Claude Vision as a third VLM reference alongside Gemini.

---

## Session Metadata
- **Date:** 2026-04-13
- **Model:** Claude Sonnet 4.6 (claude-sonnet-4-6)
- **Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`
- **Key packages used:** `pandas`, `openpyxl`, `requests`, `opencv-python`, `easyocr`, `torch`, `ultralytics/yolov5` (torch hub), `google-genai 0.x`, `reportlab`, `numpy`
- **Python version:** 3.12.9
- **torch version:** 2.11.0
- **Gemini model:** `gemini-2.5-pro`
- **Platform:** macOS Darwin 25.3.0 (CPU only)
