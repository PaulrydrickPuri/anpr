# ANPR Accuracy Analysis — Client A

> Automated pipeline to audit and benchmark an ANPR system's plate-reading accuracy using three independent methods: EasyOCR, Gemini Vision (ground truth), and a custom YOLOv5 character-detection model.

---

## Project Overview

This project evaluates the accuracy of a deployed Automatic Number Plate Recognition (ANPR) system at Client A. The client provided an Excel report (`ACCURACY REPORT.xlsx`) with 1,022 plate reading events flagged as TRUE/FALSE. This pipeline focuses on the 63 FALSE rows — downloading their plate crop images from the live server, running three independent reading methods, and producing a side-by-side comparison report to identify whether failures are caused by the model itself, the preprocessing pipeline, or image quality.

---

## Progress

| Date | Session | What Was Done |
|---|---|---|
| 2026-04-13 | [anpr-accuracy-analysis-pipeline](sessions/2026-04-13_anpr-accuracy-analysis-pipeline.md) | Built full pipeline; 63 images downloaded; EasyOCR + Gemini Vision + YOLOv5 run; Excel + HTML reports generated; YOLO outperforms deployed system on FALSE rows |
| 2026-04-13 | [human-gt-gemini25-pdf-reports](sessions/2026-04-13_human-gt-gemini25-pdf-reports.md) | Integrated human ground truth CSV; upgraded to Gemini 2.5 Pro; generated YOLO bbox PDF + Gemini Vision PDF; final accuracy: Gemini 80%, YOLO 35%, System 2.5% vs human GT |

---

## Dataset

| Item | Detail |
|---|---|
| Source | Client-provided Excel report (`ACCURACY REPORT.xlsx`) |
| Total rows | 1,022 ANPR events |
| FALSE rows analysed | 63 |
| Image source | `https://<redacted-client-image-server>/` (self-signed cert) |
| Image type | Plate crop JPEGs |
| Failure categories | MISDETECTION (33), GLITCHY CAMERA (9), WRONG LANE DETECTION (8), JPJ NOT STANDARD (4), WRONG LANE (4), others (5) |

---

## Model

| Item | Detail |
|---|---|
| Architecture | YOLOv5m |
| Task | Character-level detection (per-character bounding box + class) |
| Weights | `paul/train/train-result/weights/best.pt` |
| Classes | 46 — digits 0-9 (IDs 0-9), letters A-Z (IDs 10-35), brand logos (IDs 36-45) |
| Parameters | 21,034,779 |
| GFLOPs | 48.4 |
| Inference | CPU, ~3-4s/image |
| Status | Trained, evaluated |

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
| **YOLO best.pt** | **35.0% (14/40)** | **64.3%** |
| **Gemini 2.5 Pro** | **80.0% (32/40)** | **92.8%** |
| EasyOCR | 7.5% (3/40) | 35.3% |
| YOLO avg confidence | — | 0.751/char |

**Key insight:** Gemini 2.5 Pro (80%) is a reliable VLM ground truth. YOLO (35%) is 14× better than the deployed system (2.5%) on failed plates, confirming the production **preprocessing pipeline** — not the model weights — is the root cause of failures. 23/63 plates were marked "not clear" even by human reviewers, representing hardware/placement failures.

---

## Key Files

| File | Purpose |
|---|---|
| `sessions/` | Per-session progress logs |
| `PROGRESS.md` | Session index |
| `anpr_analysis.py` *(local)* | v1 pipeline (EasyOCR + Gemini 2.0 Flash + YOLO, HTML/Excel output) |
| `anpr_analysis_v2.py` *(local)* | v2 pipeline (human GT + Gemini 2.5 Pro + YOLO + PDF generation) |
| `report/analysis_v2/*.pdf` *(local)* | YOLO bbox annotation PDF + Gemini Vision PDF |
| `report/analysis_v2/*.csv` *(local)* | Full comparison CSV |
| `report/analysis_v2/*.xlsx` *(local)* | 4-sheet Excel workbook |
| `report/analysis_v2/annotated_yolo/` *(local)* | 126 annotation card images (63 YOLO + 63 Gemini) |

---

## Tech Stack
- **Language:** Python 3.12.9
- **OCR:** EasyOCR (English, CPU)
- **Vision Ground Truth:** Google Gemini 2.0 Flash (`google-genai` SDK)
- **Object Detection:** YOLOv5m via `torch.hub` (PyTorch 2.11.0, CPU)
- **Data:** pandas, openpyxl
- **Networking:** requests (SSL verify=False for corporate server)
- **Platform:** macOS Darwin 25.3.0

---

## Next Steps
- [ ] Investigate production preprocessing pipeline vs raw crop differences (root cause of 2.5% system accuracy)
- [ ] Fix YOLO `I` suppression — class 18 underrepresented, causes UTM/UITM confusion
- [ ] Lower NMS IOU to 0.35–0.40 to fix double-detections on wide plates
- [ ] Retrain YOLO with Putrajaya / UITM / UTEM / non-standard plate examples
- [ ] Add per-remark-category accuracy breakdown to PDF
- [ ] Package as CLI with `argparse` for client reuse
- [ ] Fund Anthropic API for Claude Vision cross-validation alongside Gemini

---

*Generated by [session-github-journal](https://github.com/PaulrydrickPuri) — Claude Code skill*
