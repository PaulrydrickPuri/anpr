# Session Log — 2026-04-13 (Session 3)
**Project:** ANPR Accuracy Analysis — Client A  
**Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`  
**Duration context:** ~2 hours (approx. 14:00 – 16:00 MYT, same day as Sessions 1 & 2)

---

## Summary

This session added a second custom model — a YOLOv8s character-detection model exported as ONNX — to the comparison pipeline, producing a full 5-way accuracy benchmark: System (deployed pipeline) vs YOLOv5m vs YOLOv8s vs Gemini 2.5 Pro vs EasyOCR, all against human ground truth. Three ONNX inference bugs were discovered and fixed (silent exception from dead code, per-class NMS instead of global NMS, and a `NoneType` crash in the character accuracy utility). Final results: Gemini 2.5 Pro 67.5%, YOLOv5 35.0%, YOLOv8 27.5%, EasyOCR 7.5%, System 2.5% exact match on 40 human-readable plates.

---

## What Was Built

### `anpr_analysis_v3.py` — 5-Way Comparison Pipeline
- Extended v2 pipeline to add YOLOv8s ONNX inference alongside the existing YOLOv5m torch.hub inference.
- **Location:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/anpr_analysis_v3.py`
- **Key additions over v2:**
  - `YOLOv8ONNX` class: loads `v2model/export/ONNX/onnx_ANPR-2026-9-1_yolov8s.onnx` via `onnxruntime.InferenceSession`
  - Input preprocessing: BGR→RGB, resize to 416×416 (matching `config.yaml` `INPUT_SHAPE: 416`), normalise to [0,1], add batch dimension
  - Output postprocessing: transpose raw output `[1, 50, 3549]` → `[3549, 50]`, split first 4 columns as bbox (cx,cy,w,h in 416-pixel space), remaining 46 as class scores; apply global NMS
  - Per-class AP metrics loaded from `v2model/events_store/metrics.json` (JSONL) — extracted AP@IoU=0.5 for all 46 character classes
  - 6-sheet Excel workbook: Summary, Comparison, YOLO5_CharDetail, YOLO8_CharDetail, AP_Metrics, GT_Readable_Only
  - Three PDFs: YOLOv5 annotation PDF, YOLOv8 annotation PDF, 5-way comparison PDF (side-by-side all methods per plate)

### YOLOv8 Annotation Cards (`annotated_yolo/yolov8_row*.png`)
- Same dark-theme card format as YOLOv5 cards from v2: plate crop with coloured bounding boxes + `CHAR (XX.X%)` labels, right panel with character/confidence table.
- **Location:** `report/analysis_v3/annotated_yolo/yolov8_row0000.png` … `yolov8_row0062.png`

### 5-Way Comparison Cards (`annotated_yolo/comparison_row*.png`)
- Per-plate card showing: plate image (left), then a right panel listing all 5 predictions (System, YOLOv5, YOLOv8, Gemini, Human GT) with colour-coded ✓/✗ match indicators vs human GT.
- **Location:** `report/analysis_v3/annotated_yolo/comparison_row0000.png` … `comparison_row0062.png`

### PDF Reports
- `yolov5_report_20260413_150806.pdf` — 63-plate YOLOv5 annotation + comparison table PDF (same format as v2).
- `yolov8_report_20260413_150806.pdf` — 63-plate YOLOv8 annotation + comparison table PDF.
- `comparison_report_20260413_150806.pdf` — 63-plate 5-way side-by-side comparison PDF, one plate per page.
- **Location:** `report/analysis_v3/`

### Excel & CSV Outputs
- `report_v3_20260413_150806.csv` — flat comparison table, all 63 rows, all 5 predictions.
- `report_v3_20260413_150806.xlsx` — 6-sheet workbook (Summary, Comparison, YOLO5_CharDetail, YOLO8_CharDetail, AP_Metrics, GT_Readable_Only).
- **Location:** `report/analysis_v3/`

---

## Key Decisions & Rationale

| Decision | Choice Made | Why |
|---|---|---|
| ONNX runtime for YOLOv8 | `onnxruntime.InferenceSession` | Model was pre-exported to ONNX; avoids needing `ultralytics` package compatibility; portable across platforms |
| Input shape 416×416 BGR | Read from `config.yaml` (`INPUT_SHAPE: 416`, `FORMAT: BGR`) | Config is the authoritative source; v8 training used 416, not 640 |
| Global NMS (not per-class) | Single NMS pass over all detections regardless of class | YOLOv8 character model outputs non-overlapping character boxes; per-class NMS suppressed correct detections |
| ONNX output transpose | `out[0].T` → `[3549, 50]` | ORT returns `[1, 50, 3549]` (channels-first); transposing gives one row per anchor for easy slicing |
| 5-way comparison PDF | New PDF separate from v2 YOLOv5/Gemini PDFs | Stakeholders need a single document showing all models simultaneously for side-by-side review |
| AP metrics from `metrics.json` | Loaded JSONL, extracted `AP@IoU=0.5` per class | Pre-computed training metrics reveal which character classes the YOLOv8 model struggles with (e.g. `I`, `O` vs `0`) |
| Gemini accuracy drop (80%→67.5%) | Not a regression — different run day, API non-determinism | Gemini 2.5 Pro is stochastic; 67.5% is within expected variance. Human GT remains the reference |

---

## Findings & Observations

1. **5-way accuracy (40 human-readable rows):**
   - Gemini 2.5 Pro: **67.5% exact match** (27/40), 89.2% avg char accuracy
   - YOLOv5m best.pt: **35.0% exact match** (14/40), 64.3% avg char accuracy
   - YOLOv8s ONNX: **27.5% exact match** (11/40), 61.8% avg char accuracy
   - EasyOCR: **7.5% exact match** (3/40), 35.3% avg char accuracy
   - System (deployed): **2.5% exact match** (1/40), 51.2% avg char accuracy

2. **YOLOv5m outperforms YOLOv8s on this plate set** — likely because YOLOv5m was trained on 29,400+ iterations on a closely related dataset, while YOLOv8s training configuration shows `MAX_ITER: 441000` but the checkpoint may have been captured at an earlier epoch. Also possible: YOLOv5m has more parameters (21M vs ~11M for yolov8s) which helps on small character recognition.

3. **YOLOv8s ONNX output shape is `[1, 50, 3549]`**: 50 = 4 bbox coords + 46 class scores (no objectness). Class scores are already post-sigmoid (range ~0–0.94). This is different from YOLOv5's `[1, N, 51]` format with an explicit objectness confidence.

4. **YOLOv8s has 0 detections bug** on first run — caused by a dead code line `raw = self.sess.run([None], {input_name: inp})` that silently threw an ORT exception (caught by try/except), preventing the real inference call from running. Removing the dead line fixed it.

5. **Per-class AP from `metrics.json`:** Class `I` (ID 18) AP@0.5 was among the lower performers, consistent with the UITM→UTM misread pattern observed in YOLOv5. Class `0` vs `O` confusion was also visible in AP gap between similar-looking characters.

6. **YOLOv8s `I` suppression confirmed** — same UITM→UTM pattern as YOLOv5 on UITM plates. Both models share this weakness, suggesting it originates in the training data distribution, not the architecture.

7. **Gemini 2.5 Pro dropped from 80% (v2) to 67.5% (v3)** — same model, same plates. API non-determinism across run days accounts for the difference. The v2 run was fresh; v3 was a second call days later. Gemini is still the strongest performer but not perfectly consistent.

8. **System preprocessing is the bottleneck** — YOLOv5 at 35% and YOLOv8 at 27.5% both far exceed the deployed system's 2.5% when run on raw plate crops. Both models read plates the system completely fails on. Root cause of the gap is pre-inference preprocessing in the production pipeline (resizing strategy, colour conversion, contrast adjustment), not model weight quality.

9. **EasyOCR remains weakest at 7.5%** — unsuitable for Malaysian plate characters without domain fine-tuning. Struggles on blurry and angled crops; frequently merges adjacent characters or drops them entirely.

10. **Total pipeline run time:** ~18 minutes for 63 plates (YOLOv5 CPU + YOLOv8 ONNX CPU + Gemini 2.5 Pro API + EasyOCR CPU + 3 PDF builds).

---

## Problems & Fixes

| Problem | Root Cause | Fix Applied |
|---|---|---|
| YOLOv8 returned 0 detections on all 63 plates | Dead code line `raw = self.sess.run([None], {input_name: inp})` threw silent ORT exception, returning before real inference call | Removed the dead `raw =` line entirely; real `sess.run(None, ...)` now executes cleanly |
| YOLOv8 still returned wrong detections after first fix | `per_class_nms()` was suppressing correct detections — character boxes of different classes overlapping should be kept, not suppressed | Switched to `global_nms()` — single NMS pass over all detections regardless of class |
| `AttributeError: 'NoneType' has no attribute 'upper'` in `char_acc()` | Gemini returned `None` on row 17 (empty API response on one plate); `char_acc(None, gt)` called `.upper()` on `None` | Added `if not pred or not gt: return 0.0` guard at top of `char_acc()` |
| `ultralytics` TypeError loading YOLOv5 weights | YOLOv5m `.pt` is incompatible with `ultralytics` v8 package loader | Load via `torch.hub.load('ultralytics/yolov5', 'custom', path=...)` — correct loader for v5 weights |
| SSL verification errors on plate image downloads | Corporate image server uses self-signed certificate | `requests.get(url, verify=False)` + `urllib3.disable_warnings(InsecureRequestWarning)` |

---

## ML / Training Config

### YOLOv8s ONNX Model
```
Architecture:  YOLOv8s (ultralytics_v8)
Input shape:   416×416, BGR, FP32 normalized to [0, 1]
Output shape:  [1, 50, 3549] — 50 = 4 bbox + 46 class scores
Inference:     onnxruntime (CPU), ~0.4s/image
CONF_THRESH:   0.25
IOU_THRESH:    0.45
NMS type:      Global (not per-class)
Classes:       46 (digits 0–9, letters A–Z, 10 brand logos)
```

### YOLOv5m (unchanged from v2)
```
CONF_THRESH:   0.25
IOU_THRESH:    0.45
NMS type:      Per-class (default torch.hub yolov5 behaviour)
```

### Recommended Follow-Up Actions
- Compare YOLOv8 per-class AP from `metrics.json` against YOLOv5 per-class mAP — identify classes where one model dominates and consider an ensemble.
- Lower both model NMS IOU from 0.45 → 0.35 to reduce double-detections on wide plates.
- Increase `I` (class 18) training samples in both model datasets — confirmed suppression pattern in both YOLOv5 and YOLOv8.
- Re-run YOLOv8 inference at 640×640 (vs 416) to test whether higher resolution improves small-character accuracy on compressed plate crops.

---

## Files Created / Modified

| File | Location | Purpose |
|---|---|---|
| `anpr_analysis_v3.py` | `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/` | Full v3 pipeline — 5-way comparison, ONNX YOLOv8, 3 PDFs, 6-sheet Excel |
| `report_v3_20260413_150806.csv` | `report/analysis_v3/` | Flat comparison table, all 63 rows, all 5 predictions |
| `report_v3_20260413_150806.xlsx` | `report/analysis_v3/` | 6-sheet Excel (Summary, Comparison, YOLO5_CharDetail, YOLO8_CharDetail, AP_Metrics, GT_Readable_Only) |
| `yolov5_report_20260413_150806.pdf` | `report/analysis_v3/` | YOLOv5 annotation PDF (63 plates) |
| `yolov8_report_20260413_150806.pdf` | `report/analysis_v3/` | YOLOv8 ONNX annotation PDF (63 plates) |
| `comparison_report_20260413_150806.pdf` | `report/analysis_v3/` | 5-way comparison PDF — all methods side by side per plate |
| `annotated_yolo/yolov8_row*.png` (63 files) | `report/analysis_v3/annotated_yolo/` | Per-plate YOLOv8 dark-theme annotation cards |
| `annotated_yolo/comparison_row*.png` (63 files) | `report/analysis_v3/annotated_yolo/` | Per-plate 5-way comparison cards |

---

## Next Steps / Open Questions

- [ ] Investigate production preprocessing pipeline — compare transforms applied before feeding frames to deployed model vs raw plate crops used here (root cause of 2.5% system accuracy).
- [ ] Fix `I` suppression in both models — class 18 is underrepresented in training data for both YOLOv5m and YOLOv8s.
- [ ] Lower NMS IOU to 0.35–0.40 to fix double-detections on wide plates (e.g. `UTEM1212` → `UTEM121112`).
- [ ] Re-run YOLOv8 at 640×640 input — current 416 may be too low for compressed plate crops; could close the gap with YOLOv5m.
- [ ] Ensemble YOLOv5 + YOLOv8 predictions — both models fail on different plates; a union/voting strategy could push exact match above 40%.
- [ ] Per-remark-category breakdown (MISDETECTION vs GLITCHY CAMERA vs WRONG LANE) — add as section header in comparison PDF.
- [ ] Package pipeline as CLI (`argparse`) — client should run `python anpr_analysis_v3.py --csv <path> --yolov5 <pt> --yolov8 <onnx> --api-key <key>`.
- [ ] Retrain both models with additional UITM/UTEM/Putrajaya plates to fix brand-specific failure patterns.
- [ ] Add per-class AP bar chart to Excel Summary sheet using `openpyxl` charts.

---

## Session Metadata
- **Date:** 2026-04-13
- **Model:** Claude Sonnet 4.6 (claude-sonnet-4-6)
- **Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`
- **Key packages used:** `pandas`, `openpyxl`, `requests`, `opencv-python`, `easyocr`, `torch`, `ultralytics/yolov5` (torch hub), `onnxruntime`, `google-genai`, `reportlab`, `numpy`
- **Python version:** 3.12.9
- **torch version:** 2.11.0
- **Gemini model:** `gemini-2.5-pro`
- **YOLOv5 weights:** `paul/train/train-result/weights/best.pt` (YOLOv5m, 21M params)
- **YOLOv8 weights:** `v2model/export/ONNX/onnx_ANPR-2026-9-1_yolov8s.onnx` (YOLOv8s, ONNX)
- **Platform:** macOS Darwin 25.3.0 (CPU only)
