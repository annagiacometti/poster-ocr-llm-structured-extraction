# Poster OCR + LLM Structured Extraction

Experimental two-stage pipeline for extracting structured metadata (`festival_name`, `city_name`, `start_date`, `end_date`, `duration_days`) from heterogeneous event poster images.

## Pipeline Architecture
1. **OCR Stage (`EasyOCR`)**: Extracts raw text blocks (IT/EN) from poster images, joining bounding-box outputs with a ` | ` delimiter to preserve layout separation.
2. **Semantic Structuring (`Qwen/Qwen2.5-3B-Instruct`)**: Maps noisy OCR text into a strict 5-key JSON schema using deterministic decoding (`do_sample=False`) and explicit `null` fallback rules for missing fields.
3. **Deterministic Post-Processing (`Python`)**: Computes inclusive event duration (`(end_date - start_date) + 1`) directly from extracted ISO dates (`YYYY-MM-DD`), comparing deterministic arithmetic against direct LLM duration estimation.
4. **Evaluation (`Excel`)**: 1-to-1 comparison against a manually annotated ground truth dataset ($N = 20$ posters) using `poster_id` as primary key.

## Evaluation Summary ($N = 20$)
* **JSON Syntactic Validity:** 20 / 20 (100.0%)
* **`city_name`:** 17 / 20 (85.0%)
* **`start_date`:** 11 / 20 (55.0%)
* **`end_date`:** 11 / 20 (55.0%)
* **`festival_name`:** 8 / 20 (40.0%)
* **`duration_days` (direct LLM calculation):** 8 / 20 (40.0%)
* **`duration_days` (deterministic Python calculation):** 13 / 20 (65.0%)

## Key Findings
* **OCR Bottleneck on Decorative Typography:** Stylized display fonts cause character distortions and text fragmentation, primarily degrading event titles and date recognition.
* **Sub-title vs. Main Title Ambiguity:** Without visual hierarchy cues (font size/position), the text-only LLM occasionally extracts exhibition subtitles or URLs instead of the main event name.
* **Delegating Calendar Arithmetic to Code:** Compact 3B language models frequently miscalculate multi-month day spans even when both dates are correctly extracted; delegating duration arithmetic to Python raises accuracy from 40% to 65%.
