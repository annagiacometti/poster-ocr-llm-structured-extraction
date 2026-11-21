# Poster OCR + LLM Structured Extraction

Experimental two-stage pipeline for extracting structured metadata (`festival_name`, `city_name`, `start_date`, `end_date`, `duration_days`) from heterogeneous event poster images.

## Dataset & Sample Size
* **Sample Size:** 20 event poster images (`poster_id` used as unique primary key).
* **Ground Truth:** Manually annotated reference dataset (`data/poster_ground_truth.xlsx`).

## Pipeline Architecture
1. **OCR Stage (`EasyOCR`)**: Extracts raw text blocks (IT/EN) from poster images, joining bounding-box outputs with a ` | ` delimiter to preserve layout separation.
2. **Semantic Structuring (`Qwen/Qwen2.5-3B-Instruct`)**: Maps noisy OCR text into a strict 5-key JSON schema using deterministic decoding (`do_sample=False`) and explicit `null` fallback rules for missing fields.
3. **Deterministic Post-Processing (`Python`)**: Computes inclusive event duration directly from extracted ISO dates (`YYYY-MM-DD`), comparing deterministic arithmetic against direct LLM duration estimation.
4. **Evaluation (`Excel`)**: 1-to-1 comparison against the ground truth dataset (`data/poster_evaluation_comparison.xlsx`).

## Evaluation Criteria & Inclusive Duration Rule
* **Inclusive Duration Rule:** Event duration is calculated inclusively so that the start day counts as Day 1:

  `duration_days = (end_date - start_date) + 1`

  *(e.g., an event running from October 6 to October 8 lasts `8 - 6 + 1 = 3` days; a single-day event lasts `1` day).*
* **Field-by-Field Matching Criteria (`1` = match, `0` = mismatch):**
  * **`city_name` & `festival_name`:** Case-insensitive semantic match against ground truth (verified manually to avoid false positives/negatives from minor punctuation or formatting differences).
  * **`start_date` & `end_date`:** Exact match on full date (`DD/MM/YYYY`), including the year (`YYYY`). Missing years (`XXXX`) or hallucinated dates are marked as `0`.
  * **`duration_days`:** Exact integer match (`INT(GT) == INT(Python_calc)`) against ground truth inclusive duration.

## Evaluation Summary (N = 20)
* **JSON Syntactic Validity:** 20 / 20 (100.0%)
* **`city_name`:** 17 / 20 (85.0%)
* **`duration_days` (deterministic Python calculation):** 13 / 20 (65.0%)
* **`start_date`:** 10 / 20 (50.0%)
* **`end_date`:** 10 / 20 (50.0%)
* **`festival_name`:** 7 / 20 (35.0%)

## Most Frequent Error Categories
| Error Category | Affected Fields | Frequency / Impact | Example & Root Cause |
| :--- | :--- | :--- | :--- |
| **Decorative Typography & OCR Noise** | `festival_name`, `start_date`, `end_date`, `city_name` | High (primary cause of title & date misses) | Stylized display fonts distort characters or drop separators (e.g., *Treviso* read as `REVISO]`, *Italian Street Photo Festival* read as `STRFESHAX`). |
| **Subtitle / URL Extraction vs. Main Title** | `festival_name` | Frequent (7/20 exact main title match) | Without visual hierarchy cues (font size/layout), the text-only LLM extracts exhibition subtitles or domain names instead of the main event title (e.g., `AMatter \| of Taste` instead of *Yeast Photo Festival*). |
| **Missing or Misread Year (`XXXX`)** | `start_date`, `end_date` | Moderate (2–3 cases) | When the year is omitted on the poster or misread by OCR, strict full-date matching fails even when day and month are correct (e.g., `02/08/XXXX` vs `02/08/2026`). |
| **LLM Calendar Arithmetic Errors** | `duration_days` (direct LLM) | Eliminated by Python post-processing | Compact 3B models miscalculate multi-month spans or omit the `+ 1` inclusive offset (e.g., `2024-07-11` to `2024-11-03` estimated as `63` days instead of `116`); Python arithmetic resolves 100% of these cases whenever both dates are extracted. |
