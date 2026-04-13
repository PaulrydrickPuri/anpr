# Session Log — 2026-04-13
**Project:** ANPR Accuracy Analysis — Genting JPO  
**Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`  
**Duration context:** ~2 hours (approx. 10:19 – 11:17 MYT)

---

## Summary

The goal of this session was to build a comprehensive automated analysis pipeline that evaluates the accuracy of a deployed ANPR (Automatic Number Plate Recognition) system at Genting JPO. The pipeline filters 63 "FALSE" accuracy rows from the client's Excel report, downloads the corresponding plate crop images, runs three independent plate-reading methods (EasyOCR, Gemini 2.0 Flash Vision, and a custom YOLOv5 character-detection model), and produces a side-by-side comparison report in both Excel (4-sheet workbook) and styled HTML. By end of session, the full pipeline ran successfully end-to-end with Gemini Vision as the gold-standard ground truth, revealing that the retrained YOLO model outperforms the original ANPR pipeline on failed cases, pointing to preprocessing as the likely root cause of errors.

---

## What Was Built

### `anpr_analysis.py` — Main Analysis Pipeline
- Single-file Python pipeline (~620 lines) covering all phases: data loading, image download, OCR, Gemini Vision, YOLOv5 inference, comparison, and report generation.
- **Location:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/anpr_analysis.py`
- **Phases:**
  1. Load `ACCURACY REPORT FOR ANPR JPO.xlsx`, filter rows where `ACCURACY ? == 0.0` → 63 FALSE rows
  2. Download plate crop images from HTTPS URLs (self-signed cert server, `verify=False` required) into `report/analysis_output/images/`
  3. Run EasyOCR (English, CPU) on each image with alphanumeric allowlist
  4. Call Gemini 2.0 Flash via `google-genai` SDK for ground-truth plate reading
  5. Run YOLOv5 character detection via `torch.hub.load('ultralytics/yolov5', 'custom', ...)`, sort detections left-to-right by x-centre, assemble plate string
  6. Compute exact match and character-level accuracy for all three comparison pairs
  7. Write Excel workbook (Summary, Comparison, YOLO_CharDetail, Raw_Results sheets) and a styled HTML report with metric cards, row table, and per-char confidence statistics

### Output Reports
- **Location:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/report/analysis_output/`
- `anpr_analysis_20260413_110244.xlsx` — 54 KB, 4-sheet workbook
- `anpr_analysis_20260413_110244.html` — 51 KB, styled browser report
- `images/` — 63 downloaded plate crop JPEGs (all 63 downloaded successfully)

---

## Key Decisions & Rationale

| Decision | Choice Made | Why |
|---|---|---|
| Ground truth source | Gemini 2.0 Flash (via `google-genai` SDK) | Anthropic API key had no credits; Gemini free tier is generous (15 RPM, no card), and `gemini-2.0-flash` has strong OCR capability |
| YOLOv5 loader | `torch.hub.load('ultralytics/yolov5', 'custom', ...)` | `best.pt` is a YOLOv5 model; the `ultralytics` pip package only supports YOLOv8+ and raises `TypeError` on YOLOv5 weights |
| SSL verification | `requests.get(..., verify=False)` + `urllib3.disable_warnings()` | Image server at `tapway-genting-jpo-viewer.edgev2.gotapway.com` uses a self-signed/unverifiable certificate |
| Character assembly | Sort YOLO detections by x-centre, concatenate `name` field | Plates read left-to-right; raw YOLO output is unordered by default |
| Char class filter | Only class IDs 0–35 (digits 0-9, letters A-Z) | Classes 36–45 are vehicle brand logos (BAMbee, Chancellor, Perodua, etc.) and should not appear in the plate string |
| Fallback ground truth | EasyOCR if Gemini key not available | Allows pipeline to run and produce partial results even without an API key |
| Output formats | Excel (4 sheets) + HTML | Excel for client delivery; HTML for quick visual review in browser |

---

## Findings & Observations

1. **63 FALSE rows** out of 1,022 total (6.2% failure rate). Remark breakdown: MISDETECTION (33), GLITCHY CAMERA (9), WRONG LANE DETECTION (8), JPJ NOT STANDARD (4), WRONG LANE (4), BROKEN PLATE (1), HALF PLATE (1), BLOCKED (1), NOT STANDARD (1), JPJ STANDARD (1).
2. **YOLO outperforms the deployed system on FALSE rows**: YOLO vs Gemini GT = 20.6% exact match / 47.3% avg char accuracy, vs System vs Gemini GT = 1.6% / 38.1%. This is the most significant finding — the retrained model is better, but something upstream is degrading its in-production performance.
3. **System vs YOLO agreement is only 4.8%** — they almost never agree on failed plates, suggesting the preprocessing or image pipeline fed to the production model differs substantially from raw plate crops.
4. **EasyOCR is unreliable for Malaysian plates** — only 7.9% exact match vs Gemini, and many empty results on glitchy/partial plates.
5. **YOLO consistently drops `I`** on UITM/UTM plates — reads `UTM447` for `UITM447`, `UTM7981` for `UITM7981`, `UTM5535` for `UITM5535`. This is an `I` vs `1` confusion in the model's training data.
6. **Gemini correctly identifies Putrajaya special plates** — reads `PUTRAJAYA8220` for the `PPutrajaya8220` entry; the system logged it with mixed case due to the branded prefix.
7. **YOLOv5 model has 46 classes**: digits (0-9, IDs 0-9), letters (A-Z, IDs 10-35), and brand logos (BAMbee, Chancellor, Malaysia, Perodua, Persona, Proton, Putra, Putrajaya, Satria, Rimau — IDs 36-45).
8. **Average YOLO confidence per character: 0.799** across 380 total detections (avg 6.03 chars/plate). Confidence is reasonably high, meaning detection confidence isn't the bottleneck — character identity confusion is.
9. **WRONG LANE / GLITCHY CAMERA cases are unsalvageable** — all three methods produce random/partial results, confirming image quality as the root cause for those categories, not model accuracy.
10. **Images are served from a corporate server** requiring SSL bypass — this should be noted for any automation deployed to a clean environment.

---

## Problems & Fixes

| Problem | Root Cause | Fix Applied |
|---|---|---|
| `ultralytics` YOLO loader rejected `best.pt` with `TypeError` | `best.pt` is YOLOv5 format; `ultralytics` pip package only handles YOLOv8+ weights | Switched to `torch.hub.load('ultralytics/yolov5', 'custom', path=...)` |
| All image downloads failed with `SSLCertVerificationError` | Corporate image server uses a self-signed / unverifiable TLS certificate | Added `verify=False` to `requests.get()` and suppressed warnings with `urllib3.disable_warnings()` |
| `openpyxl` and `pandas` not installed | Clean Python environment | `pip install openpyxl pandas` |
| `google.generativeai` (old SDK) flagged as deprecated | Anthropic had migrated to `google-genai` package | Installed `google-genai`, used `google.genai.Client` with `types.Part.from_bytes()` |
| Anthropic API key returned "credit balance too low" | Account had no credits despite key being valid | Switched ground truth provider to Google Gemini 2.0 Flash (free tier) |
| Claude Vision columns showed `(empty)` silently | `BadRequestError` was caught and swallowed; credit error surfaced only in debug run | Added direct API test before full pipeline run to surface errors early |
| `easyocr` not installed | Not in base environment | `pip install easyocr` (was already present; confirmed) |

---

## ML / Training Config

### YOLOv5 Inference Settings (applied in pipeline)
```python
model.conf = 0.25   # confidence threshold — filters low-quality detections
model.iou  = 0.45   # NMS IOU threshold — removes duplicate bounding boxes
```
- Running on CPU (no GPU detected in environment); inference is ~3-4s per image.
- Model: `YOLOv5m` — 212 layers, 21,034,779 parameters, 48.4 GFLOPs.
- Input: raw BGR plate crop from disk, converted to RGB before passing to model.

---

## Files Created / Modified

| File | Location | Purpose |
|---|---|---|
| `anpr_analysis.py` | `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/` | Main pipeline script — filter, download, OCR, Gemini, YOLO, report |
| `anpr_analysis_20260413_110244.xlsx` | `report/analysis_output/` | 4-sheet Excel comparison workbook (final run with Gemini GT) |
| `anpr_analysis_20260413_110244.html` | `report/analysis_output/` | Styled HTML report with metric cards and per-char YOLO stats |
| `anpr_analysis_20260413_101934.xlsx` | `report/analysis_output/` | Earlier run with EasyOCR as GT (pre-Gemini) |
| `anpr_analysis_20260413_101934.html` | `report/analysis_output/` | Earlier HTML run |
| `images/*.jpg` (63 files) | `report/analysis_output/images/` | Downloaded plate crop images for all 63 FALSE rows |

---

## Next Steps / Open Questions

- [ ] Re-run pipeline with a funded Anthropic API key to get Claude Vision as a second ground-truth reference (cross-validate against Gemini)
- [ ] Investigate preprocessing pipeline in the production system — what transforms are applied to images before feeding to YOLO? Compare against raw crops used in this analysis.
- [ ] Analyse the `I` vs `1` confusion in the YOLO model — check training data distribution for class `I` (ID 18) and `1` (ID 1) on UITM plates.
- [ ] Produce a per-remark-category breakdown (e.g., MISDETECTION vs GLITCHY CAMERA vs WRONG LANE) to understand which failure mode is addressable vs hardware-constrained.
- [ ] Consider fine-tuning the YOLO model with additional UITM/Putrajaya/non-standard plate examples to improve accuracy on special plate formats.
- [ ] Wrap pipeline into a CLI with `argparse` so client can re-run on new data exports without code changes.
- [ ] Add bounding-box visualisation overlays — draw YOLO detections on plate images and save annotated versions for the client report.

---

## Session Metadata
- **Date:** 2026-04-13
- **Model:** Claude Sonnet 4.6 (claude-sonnet-4-6)
- **Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`
- **Key packages used:** `pandas`, `openpyxl`, `requests`, `opencv-python`, `easyocr`, `torch`, `ultralytics/yolov5` (torch hub), `google-genai`, `anthropic` (tested, not used due to billing)
- **Python version:** 3.12.9
- **torch version:** 2.11.0
- **Platform:** macOS Darwin 25.3.0 (CPU only)
