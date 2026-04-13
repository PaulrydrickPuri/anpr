# Progress Log Index

| Date | Session | Summary |
|---|---|---|
| 2026-04-13 | [anpr-accuracy-analysis-pipeline](sessions/2026-04-13_anpr-accuracy-analysis-pipeline.md) | Built full 3-way accuracy pipeline (EasyOCR + Gemini Vision + YOLOv5) on 63 FALSE rows; YOLO beats deployed system at 20.6% vs 1.6% exact match vs Gemini GT |
| 2026-04-13 | [human-gt-gemini25-pdf-reports](sessions/2026-04-13_human-gt-gemini25-pdf-reports.md) | Added human ground truth CSV; upgraded to Gemini 2.5 Pro; built YOLO bbox annotation PDF + Gemini prediction PDF; Gemini 80% / YOLO 35% / System 2.5% exact match vs human GT |
| 2026-04-13 | [yolov8-onnx-model-comparison](sessions/2026-04-13_yolov8-onnx-model-comparison.md) | Added YOLOv8s ONNX model to pipeline; fixed 3 ONNX inference bugs; 5-way comparison (System 2.5% / YOLOv5 35% / YOLOv8 27.5% / Gemini 67.5% / EasyOCR 7.5%) vs human GT |
