# OCR Engine Comparison on a Scanned Mortgage PDF

Comparing **Tesseract**, **PaddleOCR**, and **EasyOCR** on the same scanned mortgage document — evaluating text extraction accuracy, bounding box quality, layout preservation, and suitability for downstream AI workflows.

---

## 🔍 Engines Compared

| Engine | Version | 
|---|---|
| Tesseract | Latest |
| PaddleOCR | 3.3.0 | 
| EasyOCR | Latest | 

---

## 📄 Document Used

A scanned residential mortgage lender fees worksheet — containing structured multi-column form fields, financial figures, and dense fine-print sections representative of real mortgage processing workloads.

---

## ⚙️ Setup

### Dependencies
```bash
# Tesseract
apt install tesseract-ocr
pip install pymupdf pytesseract opencv-python pillow

# PaddleOCR
pip install paddlepaddle==3.2.0
pip install paddleocr==3.3.0

# EasyOCR
pip install easyocr

# PDF utilities
pip install pdf2image
apt-get install poppler-utils
```

### Known Setup Issues & Fixes

**1. `ModuleNotFoundError: langchain.docstore`**
PaddleOCR 3.x pulls in `paddlex` which internally imports a LangChain submodule removed in LangChain 0.1+. Fix by mocking it before any imports:
```python
import sys
from unittest.mock import MagicMock

for _mod in [
    'langchain.docstore',
    'langchain.docstore.document',
    'langchain.text_splitter',
    'langchain_community',
]:
    sys.modules[_mod] = MagicMock()
```

**2. `ValueError: Unknown argument: use_gpu`**
`use_gpu` was removed in PaddleOCR 3.x. Use:
```python
ocr = PaddleOCR(use_textline_orientation=True, lang='en')
```

**3. Single letters instead of words**
PaddleOCR silently auto-downscales images exceeding 4000px, corrupting character detection. Pre-resize before saving:
```python
max_side = 3000
w, h = img_pil.size
if max(w, h) > max_side:
    scale = max_side / max(w, h)
    img_pil = img_pil.resize((int(w * scale), int(h * scale)), Image.LANCZOS)
img_pil.save(image_path, "PNG")
```

**4. `TypeError: 'module' object is not callable`**
Use the full display import path:
```python
from IPython.display import display  # not: from IPython import display
```

---

## 📊 Results

| Criterion | Tesseract | EasyOCR | PaddleOCR |
|---|---|---|---|
| Output granularity | Word-level | Segment-level | Line-level |
| Native confidence scores | No | Yes | Yes |
| Layout preservation | Poor | Good | Excellent |
| Multi-column handling | Breaks | Good | Excellent |
| Post-OCR cleanup needed | Yes | Minimal | Minimal |
| Setup difficulty | Easy | Easy | Hard |
| AI/Downstream readiness | Low | Medium | High |

---

## 📝 Findings

### Which engine worked best?
PaddleOCR produced the most complete and accurate output. Every key mortgage field was returned as a clean, intact line. The multi-column layout that fragmented Tesseract's output was handled correctly, and the output is immediately usable for downstream AI workflows without additional post-processing.

### What was surprising?
PaddleOCR silently broke character recognition when the input image exceeded 4000px on any side — it auto-downscaled internally and returned single letters instead of words with no warning. Pre-resizing to 3000px before saving resolved the issue entirely.

### Which engine suits mortgage workflows best?
PaddleOCR, once the version compatibility issues are resolved. Its line-level output maps directly to mortgage form fields and the `rec_texts` list is immediately usable when building AI pipelines on top of the extracted data.

### Frustrations with setup or output structure?
- `use_gpu=False` raised `ValueError: Unknown argument` in PaddleOCR 3.x
- `langchain.docstore.document` import crashed on load due to paddlex pulling in a removed LangChain submodule
- The 4000px auto-resize silently corrupted OCR output before the pre-resize fix was applied
- PaddleOCR 3.x changed its output format from a list of `[box, [text, score]]` pairs to a dict with `rec_texts`, `rec_scores`, and `rec_polys` keys — requiring defensive parsing

### One engine or combine for production?
**Combined approach:**
- **PaddleOCR** as the primary extractor for layout fidelity
- **EasyOCR** confidence filtering to flag uncertain fields (anything below 0.85) for manual review
- **Tesseract's** JSON bounding-box output as a cross-reference for validation on key fields

**Single engine pick:** PaddleOCR for output quality.

---

## 📁 Repository Contents

| File | Description |
|---|---|
| `Compare_3_OCR_Engines_on_a_Mortgage_PDF.ipynb` | Full Colab notebook with all three engines |
| `LenderFeesWorksheetNew.pdf` | Source mortgage document used for testing |
| `page_1.png` | Preprocessed page image used by PaddleOCR |
| `easyocr_doc.png` | Page image used by EasyOCR |

---

## 🔗 Notebook

Open directly in Google Colab:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/jshaam7/ocr-engine-comparison/blob/main/Compare_3_OCR_Engines_on_a_Mortgage_PDF.ipynb)
