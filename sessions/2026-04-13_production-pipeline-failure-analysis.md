# Session Log — 2026-04-13 (Session 4)
**Project:** ANPR Accuracy Analysis — Client A  
**Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`  
**Duration context:** ~2 hours (approx. 16:00 – 18:00 MYT, same day as Sessions 1–3)

---

## Summary

This session replicated the deployed system's production inference and postprocessing pipeline on the 63 FALSE plate images using both YOLOv5m and YOLOv8s, to understand exactly why the system achieves only 2.5% accuracy on its own failed plates. A standalone runner (`run_production_pipeline.py`) was built that faithfully mirrors the production code from `inference/object_detection_yolov5.py` and `postprocessing/postprocess_anpr.py`, including all NMS settings sourced from `inference_config.yaml` (conf=0.5, iou=0.5, input=416), the letterbox+resize double transform, the merge-NMS secondary pass, and the pad filter. YOLOv8s + production postprocessing achieved exactly 2.5% — confirming it is the deployed model. A systematic failure mode analysis then identified that **65% of failures are caused by the postprocessing chain** (`get_sequence` and `correct_regex`), not the model weights.

---

## What Was Built

### `run_production_pipeline.py` — Production Replication Runner
- Standalone script that imports `inference/` and `postprocessing/` directly and runs the full production chain on all 63 plate images.
- **Location:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/run_production_pipeline.py`
- **Key design decisions:**
  - `tensorrt` mocked with a stub module (`trt_mock.__spec__` set via `importlib.util.spec_from_loader`) to allow `postprocess_anpr.py` to import on macOS without CUDA
  - YOLOv5 loaded via `torch.load(..., weights_only=False)` with hub cache path added to `sys.path` to resolve `models.yolo.DetectionModel`
  - YOLOv8 loaded via `onnxruntime.InferenceSession` — same as v3 pipeline
  - Production preprocessing replicated exactly: `letterbox` → `cv2.resize(416,416)` → BGR→RGB → `/ 255.0` — matching `img_transform()` in production code
  - Primary NMS: `conf=0.5, iou=0.5` (from `inference_config.yaml`), `agnostic=True, multi_label=False`
  - Secondary (merge) NMS: `conf=0.3, iou=0.15` (hardcoded in production `__call__`)
  - `merge_nms` reimplemented without JIT `box_iou` (plain Python IoU to avoid `[1,1]` tensor scalar issue)
  - **Pad filter** applied after NMS: removes detections with bbox coordinates inside the letterbox padding region — mirrors production `filter padxy` step
  - `get_sequence` → `correct_regex` → `vanity_logo_Checker` postprocessing chain applied in sequence
- **Outputs:** CSV + 4-sheet Excel (Summary, Comparison, GT_Readable_Only, PostProc_Changes) in `report/analysis_prod/`

### Failure Mode Analysis
- Python analysis script comparing raw detections vs simple-sort (v3) vs production postproc vs human GT.
- Classified every readable-GT failure into one of four categories: GET_SEQUENCE_REORDER, REGEX_CORRECTION_HURT, CHAR_CONFUSION, MISSING_CHARS.
- Showed that the same raw detections give 35% (simple) vs 5% (production) for YOLOv5, isolating the postprocessing as the damage source.

---

## Key Decisions & Rationale

| Decision | Choice Made | Why |
|---|---|---|
| NMS parameters source | `yolov5m/train/inference_config.yaml` | Authoritative deployment config: `confidence: 0.5`, `image_size: 416`, `iou_threshold: 0.5` |
| Input shape | 416×416 (changed from 512×512 in earlier runs) | `inference_config.yaml` confirmed 416; YOLOv8 config also 416 |
| tensorrt mock | `importlib.util.spec_from_loader` stub | torchvision's `trace_rules.py` calls `find_spec()` which raises `ValueError` if `__spec__` is None |
| Plain Python IoU in merge | Replaced JIT `box_iou` with inline numpy | `@torch.jit.script box_iou` returns `[1,1]` tensor; `if tensor >= 0.8` fails with "Boolean value ambiguous" |
| YOLOv5 loading path | Added `~/.cache/torch/hub/ultralytics_yolov5_master` to `sys.path` | `torch.load` needs `models.yolo.DetectionModel` in Python path; hub cache has it |
| Pad filter inclusion | Added post-NMS letterbox pad filter | Was present in production `__call__` but missing from earlier runs; needed to replicate system exactly |
| Deployed model identified as YOLOv8s | YOLOv8 + prod = 2.5%, system = 2.5% | YOLOv5 + same prod pipeline = 5% — only YOLOv8 matches exactly |

---

## Findings & Observations

1. **YOLOv8s + production postprocessing = 2.5% exact match, same as deployed system.** This confirms the deployed system uses YOLOv8s (v2model), not YOLOv5m. YOLOv5m + same pipeline gives 5%.

2. **65% of failures are caused by production postprocessing, not the model.** The model detects correct characters on most plates; the postprocessing chain corrupts them before output.

3. **`get_sequence` reorders correct detections on 12/40 readable plates (30%).** It was designed for full-frame video detection with potentially tilted plates. On pre-cropped plate images, the Hough slope angle calculation with 6–8 character bounding box centres is unstable and produces wrong orderings. Examples:
   - `G1M7126` (raw) → `G6217M` (prod)
   - `JRH7379` (raw) → `JB737HR` (prod)
   - `UTM7981` (raw) → `UT897MT` (prod)
   - `QS8856Q` (raw) → `QQ6588S` (prod)

4. **`correct_regex` applies wrong alpha↔numeric substitutions on 14/40 plates (35%).** The function applies positional rules designed for standard MY plates (e.g. "first char must be alpha → replace `8` with `B`"). On plates that don't fit the expected format exactly, it corrupts correct raw detections. Examples:
   - `VMC8103` → `VB018CM` (`8→B`, then reorder)
   - `VKY1693` → `VKY6193` (digit position flip)
   - `UTM5535` → `US355MT` (reorder + substitution)

5. **Character confusion at detection level: 6/40 plates (15%).** The model detects the right set of characters but in the wrong spatial order even before `get_sequence`. Most cases have correct character set but NMS bbox centres are close together, causing x-sort ambiguity (e.g. `9A7L8A8` for `ALA9788`). One case (`AMX6702` → `AHX6702`) is a genuine `M`→`H` model confusion.

6. **Missing characters: 3/40 plates (8%).** At `conf=0.5`, partially occluded or low-contrast characters are suppressed. The secondary merge NMS at `conf=0.3` was meant to recover these but the pad filter then removes some from borderline positions.

7. **Deployed system pipeline replication required 6 bug fixes to get right.** The most impactful were: wrong input shape (512→416), wrong confidence (0.35→0.5), missing pad filter, and `box_iou` JIT tensor scalar issue.

8. **`inference_config.yaml` (`yolov5m/train/inference_config.yaml`) is the ground truth for deployment parameters.** Confirmed: `confidence: 0.5`, `image_size: 416`, `iou_threshold: 0.5`.

9. **Simple left-to-right x-sort (v3 pipeline, 35%) vs production postprocessing (5%)** — a 30 percentage point drop from the same raw detections. The entire gap between what the model can do and what the deployed system delivers is the postprocessing chain.

10. **Both models detect the correct characters on far more plates than the system reports.** YOLOv5m raw detections match GT on many plates that the production postproc then corrupts. The model weights are sound; the postprocessing is not appropriate for plate-crop inputs.

---

## Problems & Fixes

| Problem | Root Cause | Fix Applied |
|---|---|---|
| `ValueError: tensorrt.__spec__ is None` on import | torchvision `trace_rules.py` calls `find_spec(import_name)` which requires `__spec__` on all sys.modules entries | Set `trt_mock.__spec__ = importlib.util.spec_from_loader("tensorrt", loader=None)` before any torch import |
| `ModuleNotFoundError: No module named 'models'` loading YOLOv5 | `torch.load` on `.pt` unpickles `models.yolo.DetectionModel` which requires the YOLOv5 source on `sys.path` | Added `~/.cache/torch/hub/ultralytics_yolov5_master` to `sys.path` before `torch.load` |
| `UnpicklingError: weights_only load failed` | PyTorch 2.6 changed `weights_only` default to `True`; YOLOv5 `.pt` contains non-tensor objects | `torch.load(..., weights_only=False)` |
| `ValueError: 'tuple' object has no attribute 'shape'` in NMS | Raw YOLOv5 model returns `(inference_tensor, train_outputs)` tuple; NMS expects just the tensor | `result = raw_result[0] if isinstance(raw_result, (tuple, list)) else raw_result` |
| `a Tensor with 4 elements cannot be converted to Scalar` in merge | `@torch.jit.script box_iou` returns `[1,1]` tensor; `if iou >= 0.8` fails on non-scalar bool | Replaced with plain Python IoU function `_iou_1d()` |
| `IndexError: list index out of range` in `get_sequence` | Function requires ≥2 detections to initialise slope; crashes on 1-character plates | Guard: `if len(seq_data) < 2: return simple_x_sort` |
| `IndexError: string index out of range` in `vanity_checker` | `lp_str[-2]` called on plates with 0 or 1 characters | Guard: `if len(lp_str) >= 2: try vanity_checker else pass` |
| YOLOv5 giving 5% instead of matching system's 2.5% | Wrong input shape (512), wrong conf (0.35), wrong iou (0.35), missing pad filter | Updated from `inference_config.yaml`: shape=416, conf=0.5, iou=0.5; added `yolov5_pad_filter()` step |
| `FileNotFoundError: paul/train/train-result/weights/best.pt` | Path from session memory was wrong; actual directory is `yolov5m/` | Updated `YOLOV5_PT` path to `yolov5m/train/train-result/weights/best.pt` |
| `Loaded 0 FALSE rows` | CSV `ACCURACY ?` column contains Python bool `False`, not string `"FALSE"` | `.astype(str).str.strip().str.lower() == "false"` |

---

## ML / Training Config

### Deployed System — Confirmed Parameters (from `inference_config.yaml`)
```yaml
# yolov5m/train/inference_config.yaml
confidence: 0.5
image_size: 416
iou_threshold: 0.5
```

### Production NMS Chain (from `object_detection_yolov5.py`)
```python
# Primary NMS
out = non_max_suppression(result, conf_thres=0.5, iou_thres=0.5,
                          agnostic=True,   # class-agnostic (ANPR mode)
                          multi_label=False)  # best class only

# Secondary (merge) NMS — recovers missed chars
lower_out = non_max_suppression(result, conf_thres=0.3, iou_thres=0.15,
                                agnostic=True, multi_label=False)
out = merge_nms(lower_out, out)

# Pad filter — removes detections in letterbox padding region
out = [det for det in out if inside_active_region(det, pad_x, pad_y)]
```

### Recommended Fix for Production
- Replace `get_sequence` with simple `sorted(dets, key=lambda d: d[0])` for plate-crop inputs — this alone would recover ~30% of accuracy
- Apply `correct_regex` only when the raw sequence does not already match a valid MY plate pattern — skip it when the detected string is already valid
- Lower confidence to 0.35–0.40 to recover missed characters (offset by better NMS tuning)

---

## Files Created / Modified

| File | Location | Purpose |
|---|---|---|
| `run_production_pipeline.py` | `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO/` | Full production pipeline replication — inference + postprocessing for both YOLOv5m and YOLOv8s |
| `prod_pipeline_20260413_164759.csv` | `report/analysis_prod/` | First run output (512×512, conf=0.35 — superseded) |
| `prod_pipeline_20260413_170206.csv` | `report/analysis_prod/` | Final corrected run (416×416, conf=0.5) |
| `prod_pipeline_20260413_170206.xlsx` | `report/analysis_prod/` | 4-sheet Excel: Summary, Comparison, GT_Readable_Only, PostProc_Changes |

---

## Next Steps / Open Questions

- [ ] Fix `get_sequence` for plate-crop mode — use simple x-sort instead of slope-based sort when input is a pre-cropped plate image (not full frame)
- [ ] Fix `correct_regex` — skip corrections when the raw detected string already matches a valid MY plate pattern
- [ ] Lower deployed confidence threshold from 0.5 → 0.35 to recover missed characters
- [ ] Investigate the 6 char-confusion rows — 5 of 6 have the right characters but wrong order (NMS bbox overlap issue); 1 is a genuine `M`→`H` model confusion
- [ ] Verify whether the deployed system runs inference on raw plate crops or full-frame crops — the preprocessing difference may compound the postprocessing issue
- [ ] Run the corrected postprocessing (simple sort, no regex) on all 63 plates to quantify the expected improvement before redeployment
- [ ] Push failure analysis findings to client as a written root cause report

---

## Session Metadata
- **Date:** 2026-04-13
- **Model:** Claude Sonnet 4.6 (claude-sonnet-4-6)
- **Working directory:** `/Users/paulrydrickpuri/Desktop/ANPR_Report_JPO`
- **Key packages used:** `torch`, `onnxruntime`, `opencv-python`, `pandas`, `openpyxl`, `fuzzywuzzy`, `imutils`, `rapidfuzz`, `python-Levenshtein`, `numpy`
- **Python version:** 3.12.9
- **YOLOv5m weights:** `yolov5m/train/train-result/weights/best.pt`
- **YOLOv8s ONNX:** `v2model/export/ONNX/onnx_ANPR-2026-9-1_yolov8s.onnx`
- **Inference config:** `yolov5m/train/inference_config.yaml` — `conf=0.5, iou=0.5, image_size=416`
- **Platform:** macOS Darwin 25.3.0 (CPU only)
