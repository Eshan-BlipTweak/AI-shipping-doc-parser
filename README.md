# AI Shipping Document Parser

A document intelligence pipeline that extracts structured data from scanned Bills of Lading using OCR and LLM inference.

**Pipeline:** PDF → Image (pdf2image) → OCR (Tesseract) → LLM Extraction (GPT-OSS-120B) → Structured JSON

---

## Background

During my internship at ODeX, We built a production document intelligence pipeline that processed scanned Bills of Lading at scale, achieving **97% schema fulfillment in production**. This repository contains:

1. **A simplified demo pipeline** (`shipping_doc_parser.ipynb`) — rebuilt independently to demonstrate the core extraction logic
2. **An OpenClaw shipping domain skill** (`openclaw-skill/SKILL.md`) — a conversational agent with BL domain expertise

The production pipeline we built at ODeX was significantly more complex. This demo shows the core logic.

---

## Production Pipeline (ODeX)

The full production system handled hundreds of documents per day across different carriers, formats, and scan qualities:

```
PDF
 └→ pdf2image (convert to images)
 └→ CNN Page Classifier (filter relevant BL pages, discard irrelevant)
 └→ OpenCV Preprocessing (denoise, deskew, contrast enhancement)
 └→ Bounding Box Detection (identify field regions spatially)
 └→ Tesseract OCR (spatially-aware text extraction)
 └→ Prompt Engineering (schema-aware, constrained output, fallback defaults)
 └→ Groq LLM Inference (Llama-3-8B, GPT-OSS-20B, GPT-OSS-120B — domain fine-tuned)
 └→ Schema Validation (catch hallucinations, flag low confidence)
 └→ JSON Output
 └→ FastAPI
 └→ RAG Feedback Loop (corrected JSONs stored as vector embeddings, ground future extractions)
```

**Result: 97% schema fulfillment in production. I was first to deploy it.**

### Why each component exists

| Component | Problem it solves |
|---|---|
| CNN Classifier | Filters irrelevant pages before OCR runs — saves compute, reduces noise |
| OpenCV Preprocessing | Poor scan quality, stamps, skewed pages break OCR accuracy |
| Bounding Box Detection | Shipper and Shipping Agent look identical in raw text but are spatially distinct on the document |
| Fine-tuned models | Domain-specific terminology (Incoterms, HS codes, port names) requires domain knowledge |
| Schema Validation | Catches hallucinations and flags low-confidence extractions before they reach downstream systems |
| RAG Feedback Loop | Unknown document layouts improve over time — system learns from every new carrier format |

---

## Demo Pipeline

This notebook demonstrates the core extraction logic:

```
PDF → pdf2image (convert to image) → Tesseract OCR (extract text) → GPT-OSS-120B (extract fields) → Structured JSON
```

### Setup

1. Open `shipping_doc_parser.ipynb` in Google Colab
2. Add your OpenRouter API key to Colab Secrets:
   - Click the key icon in the left sidebar
   - Add secret named `OPENROUTER_API_KEY`
   - Get a free key at [openrouter.ai](https://openrouter.ai)
3. Run all cells in order
4. Upload a Bill of Lading PDF when prompted

### Model

Uses `openai/gpt-oss-120b:free` via OpenRouter — same model family as production.

### Known Limitations

This simplified pipeline works well on clean documents but has predictable failure modes that the production system was specifically built to address:

- **Field confusion** — without spatial context, the model can confuse fields that appear similar in raw text (e.g. Shipper vs Shipping Line Agent, or picking up a value from the wrong section of the document entirely)
- **Missing values** — fields buried in stamps, handwriting, or poor scan regions may be missed by Tesseract and therefore invisible to the LLM
- **Inaccurate values** — noisy OCR output (broken characters, merged words) can cause the model to extract partial or incorrect values for fields it did find
- **No validation** — without schema validation, there is no safety net to catch hallucinations or flag low-confidence extractions before they reach downstream systems

These are not bugs — they are the exact reasons the production pipeline needed bounding box detection, OpenCV preprocessing, fine-tuned models, and schema validation on top of the core OCR + LLM approach.

---

## OpenClaw Shipping Domain Skill

The `openclaw-skill/SKILL.md` file is a domain knowledge skill for [OpenClaw](https://openclaw.ai) — an open-source local AI agent framework.

### What it does
- Answers questions about Bills of Lading and shipping terminology
- Extracts structured JSON from BL text
- Understands trade finance implications of different BL types
- Knows key parties, Incoterms, HS codes, and container types

### Setup
1. Install OpenClaw from [openclaw.ai](https://openclaw.ai)
2. Copy `SKILL.md` to your OpenClaw workspace skills folder
3. Configure your model (GPT-OSS-120B via OpenRouter recommended)
4. Start the gateway and connect via browser or Telegram

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Tesseract OCR | Text extraction from scanned PDFs |
| pdf2image / poppler | PDF to image conversion |
| GPT-OSS-120B | LLM inference for structured extraction |
| OpenRouter | Model API gateway |
| OpenClaw | Local AI agent framework |
| Google Colab | Demo environment |

---

## What I Learned

- Prompt engineering is one of the highest-leverage improvements in a document intelligence pipeline — and costs nothing in compute
- Spatial context is critical for documents with repeating field patterns (shipper vs agent)
- RAG as a feedback loop — not just retrieval — makes the system self-improving
- Schema validation is non-negotiable at scale; LLMs hallucinate on edge cases

---

## Author

**Eshan Tiwari** — [GitHub](https://github.com/Eshan-BlipTweak)

Built as part of internship work at ODeX and independently extended.
