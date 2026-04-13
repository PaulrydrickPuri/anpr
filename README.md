# ANPR Accuracy Analysis — Client A

> Automated pipeline to audit and benchmark an ANPR system's plate-reading accuracy using five independent methods: EasyOCR, Gemini 2.5 Pro (VLM ground truth), YOLOv5m, YOLOv8s ONNX, and Human Ground Truth.

---

## Project Overview

This project evaluates the accuracy of a deployed Automatic Number Plate Recognition (ANPR) system at Client A. The client provided an Excel report (`ACCURACY REPORT.xlsx`) with 1,022 plate reading events flagged as TRUE/FALSE. This pipeline focuses on the 63 FALSE rows — downloading their plate crop images from the live server, running five independent reading methods, and producing a side-by-side comparison report to identify whether failures are caused by the model itself, the preprocessing pipeline, or image quality.

---

## Progress

| Date | Session | What Was Done |
|---|---|---|
| 2026-04-13 | [anpr-accuracy-analysis-pipeline](sessions/2026-04-13_anpr-accuracy-analysis-pipeline.md) | Built full pipeline; 63 images downloaded; EasyOCR + Gemini Vision + YOLOv5 run; Excel + HTML reports generated; YOLO outperforms deployed system on FALSE rows |
| 2026-04-13 | [human-gt-gemini25-pdf-reports](sessions/2026-04-13_human-gt-gemini25-pdf-reports.md) | Integrated human ground truth CSV; upgraded to Gemini 2.5 Pro; generated YOLO bbox PDF + Gemini Vision PDF; final accuracy: Gemini 80%, YOLO 35%, System 2.5% vs human GT |
| 2026-04-13 | [yolov8-onnx-model-comparison](sessions/2026-04-13_yolov8-onnx-model-comparison.md) | Added YOLOv8s ONNX to pipeline; fixed 3 ONNX inference bugs; 5-way comparison: Gemini 67.5% / YOLOv5 35% / YOLOv8 27.5% / System 2.5% / EasyOCR 7.5% vs human GT |
| 2026-04-13 | [production-pipeline-failure-analysis](sessions/2026-04-13_production-pipeline-failure-analysis.md) | Replicated production pipeline (YOLOv5m = deployed model, confirmed); root cause: 65% of failures from postprocessing (get_sequence 30% + correct_regex 35%), not model weights |

---

## Dataset

| Item | Detail |
|---|---|
| Source | Client-provided Excel report (`ACCURACY REPORT.xlsx`) |
| Total rows | 1,022 ANPR events |
| FALSE rows analysed | 63 |
| Human GT readable | 40/63 (23 marked "not clear" by human reviewers) |
| Image source | `https://<redacted-client-image-server>/` (self-signed cert) |
| Image type | Plate crop JPEGs |
| Failure categories | MISDETECTION (33), GLITCHY CAMERA (9), WRONG LANE DETECTION (8), JPJ NOT STANDARD (4), WRONG LANE (4), others (5) |

---

## Models

### YOLOv5m — Primary Detection Model *(Deployed)*

| Item | Detail |
|---|---|
| Architecture | YOLOv5m |
| Task | Character-level detection (per-character bounding box + class) |
| Weights | `paul/train/train-result/weights/best.pt` |
| Classes | 46 — digits 0-9, letters A-Z, brand logos (IDs 36-45) |
| Parameters | 21,034,779 |
| GFLOPs | 48.4 |
| Inference | CPU via torch.hub, ~3-4s/image |
| Status | Trained, evaluated |

### YOLOv8s — ONNX Export Model

| Item | Detail |
|---|---|
| Architecture | YOLOv8s |
| Task | Character-level detection (per-character bounding box + class) |
| Weights | `v2model/export/ONNX/onnx_ANPR-2026-9-1_yolov8s.onnx` |
| Classes | 46 — same class map as YOLOv5m |
| Input shape | 416×416, BGR, FP32 normalised [0,1] |
| Output shape | [1, 50, 3549] — 4 bbox + 46 class scores per anchor |
| Inference | CPU via onnxruntime, ~0.4s/image |
| Status | Trained, ONNX exported, evaluated |

---

## Class Map (Alphanumeric — IDs 0–35)

| ID | Class | ID | Class | ID | Class |
|---|---|---|---|---|---|
| 0 | 0 | 12 | C | 24 | O |
| 1 | 1 | 13 | D | 25 | P |
| 2 | 2 | 14 | E | 26 | Q |
| 3 | 3 | 15 | F | 27 | R |
| 4 | 4 | 16 | G | 28 | S |
| 5 | 5 | 17 | H | 29 | T |
| 6 | 6 | 18 | I | 30 | U |
| 7 | 7 | 19 | J | 31 | V |
| 8 | 8 | 20 | K | 32 | W |
| 9 | 9 | 21 | L | 33 | X |
| 10 | A | 22 | M | 34 | Y |
| 11 | B | 23 | N | 35 | Z |

Brand logo classes (36–45): BAMbee, Chancellor, Malaysia, Perodua, Persona, Proton, Putra, Putrajaya, Satria, Rimau

---

## Key Results (2026-04-13, vs Human Ground Truth — 40 readable rows)

| Source | Exact Match | Avg Char Accuracy |
|---|---|---|
| System (deployed pipeline) | 2.5% (1/40) | 51.2% |
| EasyOCR | 7.5% (3/40) | 35.3% |
| **YOLOv8s ONNX** | **27.5% (11/40)** | **61.8%** |
| **YOLOv5m best.pt** | **35.0% (14/40)** | **64.3%** |
| **Gemini 2.5 Pro** | **67.5% (27/40)** | **89.2%** |

**Key insight:** Both custom YOLO models (YOLOv5m 35%, YOLOv8s 27.5%) vastly outperform the deployed system (2.5%) on the same plate crops. The 14× gap confirms the production **preprocessing pipeline** — not the model weights — is the root cause of failures. Gemini 2.5 Pro at 67.5% serves as a reliable VLM cross-check. 23/63 plates were marked "not clear" by human reviewers — excluded from all accuracy metrics.

---

## Key Files

| File | Purpose |
|---|---|
| `sessions/` | Per-session progress logs |
| `PROGRESS.md` | Session index |
| `anpr_analysis.py` *(local)* | v1 pipeline (EasyOCR + Gemini 2.0 Flash + YOLOv5, HTML/Excel output) |
| `anpr_analysis_v2.py` *(local)* | v2 pipeline (human GT + Gemini 2.5 Pro + YOLOv5 + PDF generation) |
| `anpr_analysis_v3.py` *(local)* | v3 pipeline (5-way comparison: adds YOLOv8s ONNX, 3 PDFs, 6-sheet Excel) |
| `report/analysis_v2/*.pdf` *(local)* | YOLOv5 bbox annotation PDF + Gemini Vision PDF (v2) |
| `report/analysis_v3/*.pdf` *(local)* | YOLOv5 PDF + YOLOv8 PDF + 5-way comparison PDF (v3) |
| `report/analysis_v3/*.xlsx` *(local)* | 6-sheet Excel (Summary, Comparison, YOLO5_CharDetail, YOLO8_CharDetail, AP_Metrics, GT_Readable_Only) |
| `v2model/export/ONNX/*.onnx` *(local)* | YOLOv8s ONNX weights |
| `v2model/events_store/metrics.json` *(local)* | Per-class AP metrics (JSONL) from YOLOv8s training |

---

## Tech Stack
- **Language:** Python 3.12.9
- **OCR:** EasyOCR (English, CPU)
- **Vision Ground Truth:** Google Gemini 2.5 Pro (`google-genai` SDK)
- **Object Detection:** YOLOv5m via `torch.hub` (PyTorch 2.11.0, CPU) + YOLOv8s via `onnxruntime` (CPU)
- **PDF generation:** ReportLab
- **Data:** pandas, openpyxl
- **Networking:** requests (SSL verify=False for corporate server)
- **Platform:** macOS Darwin 25.3.0

---

## Root Cause Summary (2026-04-13)

The deployed system's 2.5% accuracy on failed plates is **not a model problem** — it is a postprocessing problem:

| Failure Mode | Plates Affected | Layer |
|---|---|---|
| `correct_regex` bad alpha↔numeric substitutions | 14/40 (35%) | Postprocessing |
| `get_sequence` reorders correct detections | 12/40 (30%) | Postprocessing |
| Char confusion at detection level | 6/40 (15%) | Model |
| Missing characters (conf=0.5 too high) | 3/40 (8%) | Threshold |

**Fix:** Replace `get_sequence` with simple left-to-right x-sort for plate-crop inputs; apply `correct_regex` only when detected string does not already match a valid MY plate pattern.

---

## Next Steps
- [ ] Fix `get_sequence` for plate-crop mode — use x-sort, not slope-sort
- [ ] Fix `correct_regex` — skip when raw string already matches valid MY plate pattern
- [ ] Lower deployed confidence threshold from 0.5 → 0.35 to recover missed characters
- [ ] Run corrected postprocessing on 63 plates to quantify expected improvement before redeployment
- [ ] Fix `I` (class 18) suppression in both models — causes UITM→UTM misreads
- [ ] Verify whether deployed system runs on raw plate crops or full-frame crops
- [ ] Package pipeline as CLI (`argparse`) for client reuse
- [ ] Retrain both models with Putrajaya / UITM / UTEM / non-standard plate examples

---

*Generated by [session-github-journal](https://github.com/PaulrydrickPuri) — Claude Code skill*
