---
layout: default
title:  "AI-Powered Document Processing Pipeline"
date:   2025-11-17 18:00:00
categories: AI LLM OCR Enterprise
---

Intelligent document processing is one of the most popular enterprise AI use cases. LLMs combined with OCR can extract structured data from PDFs, invoices, contracts, and forms with high accuracy. Here's how to build a production pipeline.

## Pipeline Architecture

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Input   │───▶│  OCR /   │───▶│  LLM     │───▶│  Output  │
│  (PDF)   │    │  Parse   │    │  Extract │    │  (JSON)  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

## Approaches Compared

| Approach | Best For | Accuracy | Cost | Speed |
|----------|----------|----------|------|-------|
| Traditional OCR | Simple text | 90-95% | Low | Fast |
| LLM Vision | Complex layouts | 95-99% | Medium | Medium |
| Hybrid (OCR+LLM) | Production | 98-99%+ | Medium | Medium |

## Basic OCR Pipeline

### Using Tesseract + LLM

```python
import pytesseract
from pdf2image import convert_from_path
from openai import OpenAI
import json

client = OpenAI()

class BasicDocProcessor:
    def __init__(self):
        self.ocr_config = "--oem 3 --psm 6"

    def process_pdf(self, pdf_path: str, schema: dict) -> dict:
        """Process PDF and extract structured data."""
        # Convert PDF to images
        images = convert_from_path(pdf_path)

        # OCR each page
        text_content = []
        for i, image in enumerate(images):
            text = pytesseract.image_to_string(image, config=self.ocr_config)
            text_content.append(f"--- Page {i+1} ---\n{text}")

        full_text = "\n\n".join(text_content)

        # Extract with LLM
        return self.extract_with_llm(full_text, schema)

    def extract_with_llm(self, text: str, schema: dict) -> dict:
        """Use LLM to extract structured data."""
        prompt = f"""Extract information from this document according to the schema.

Document text:
{text[:8000]}

Schema:
{json.dumps(schema, indent=2)}

Return only valid JSON matching the schema. Use null for missing values."""

        response = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)

# Usage
processor = BasicDocProcessor()

schema = {
    "invoice_number": "string",
    "date": "string (YYYY-MM-DD)",
    "vendor_name": "string",
    "total_amount": "number",
    "line_items": [{
        "description": "string",
        "quantity": "number",
        "unit_price": "number"
    }]
}

result = processor.process_pdf("invoice.pdf", schema)
```

## Vision LLM Approach

### Direct PDF Processing with GPT-4 Vision

```python
import base64
from pdf2image import convert_from_path
from io import BytesIO

class VisionDocProcessor:
    def __init__(self):
        self.client = OpenAI()

    def pdf_to_base64_images(self, pdf_path: str) -> list[str]:
        """Convert PDF pages to base64 images."""
        images = convert_from_path(pdf_path, dpi=150)
        base64_images = []

        for image in images:
            buffer = BytesIO()
            image.save(buffer, format="PNG")
            base64_str = base64.b64encode(buffer.getvalue()).decode()
            base64_images.append(base64_str)

        return base64_images

    def process_pdf(self, pdf_path: str, schema: dict) -> dict:
        """Process PDF using vision model."""
        images = self.pdf_to_base64_images(pdf_path)

        # Build message with images
        content = [
            {
                "type": "text",
                "text": f"""Extract data from this document according to this schema:

{json.dumps(schema, indent=2)}

Return only valid JSON. Use null for missing values."""
            }
        ]

        # Add page images
        for i, img_b64 in enumerate(images[:5]):  # Limit pages
            content.append({
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/png;base64,{img_b64}",
                    "detail": "high"
                }
            })

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": content}],
            response_format={"type": "json_object"},
            max_tokens=4096
        )

        return json.loads(response.choices[0].message.content)
```

### Using Claude Vision

```python
from anthropic import Anthropic

class ClaudeDocProcessor:
    def __init__(self):
        self.client = Anthropic()

    def process_pdf(self, pdf_path: str, schema: dict) -> dict:
        """Process PDF using Claude vision."""
        images = self.pdf_to_base64_images(pdf_path)

        content = []

        # Add images
        for img_b64 in images[:5]:
            content.append({
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/png",
                    "data": img_b64
                }
            })

        # Add extraction prompt
        content.append({
            "type": "text",
            "text": f"""Extract data from these document pages.

Schema:
{json.dumps(schema, indent=2)}

Return only valid JSON."""
        })

        response = self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=4096,
            messages=[{"role": "user", "content": content}]
        )

        # Parse JSON from response
        return json.loads(response.content[0].text)
```

## Hybrid Pipeline

Combine OCR for text and Vision for layout understanding:

```python
import easyocr
from dataclasses import dataclass

@dataclass
class TextBlock:
    text: str
    bbox: tuple  # (x1, y1, x2, y2)
    confidence: float

class HybridDocProcessor:
    def __init__(self):
        self.reader = easyocr.Reader(['en'])
        self.client = OpenAI()

    def process_pdf(self, pdf_path: str, schema: dict) -> dict:
        """Hybrid OCR + LLM processing."""
        images = convert_from_path(pdf_path, dpi=200)

        all_blocks = []
        for page_num, image in enumerate(images):
            blocks = self.ocr_with_layout(image, page_num)
            all_blocks.extend(blocks)

        # Structure text with coordinates
        structured_text = self.format_with_layout(all_blocks)

        # Extract with LLM
        return self.extract_structured(structured_text, schema)

    def ocr_with_layout(self, image, page_num: int) -> list[TextBlock]:
        """OCR with bounding boxes."""
        import numpy as np

        results = self.reader.readtext(np.array(image))

        blocks = []
        for bbox, text, confidence in results:
            # Convert bbox to rectangle
            x1 = min(p[0] for p in bbox)
            y1 = min(p[1] for p in bbox)
            x2 = max(p[0] for p in bbox)
            y2 = max(p[1] for p in bbox)

            blocks.append(TextBlock(
                text=text,
                bbox=(x1, y1, x2, y2),
                confidence=confidence
            ))

        return blocks

    def format_with_layout(self, blocks: list[TextBlock]) -> str:
        """Format text preserving layout information."""
        # Sort by position (top to bottom, left to right)
        blocks.sort(key=lambda b: (b.bbox[1], b.bbox[0]))

        formatted = []
        for block in blocks:
            if block.confidence > 0.5:
                formatted.append(
                    f"[{block.bbox[0]:.0f},{block.bbox[1]:.0f}] {block.text}"
                )

        return "\n".join(formatted)

    def extract_structured(self, text: str, schema: dict) -> dict:
        """Extract using layout-aware prompt."""
        prompt = f"""Extract data from this OCR output with coordinates.
The coordinates [x,y] indicate text position on the page.

OCR Output:
{text[:10000]}

Schema:
{json.dumps(schema, indent=2)}

Return valid JSON matching the schema."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)
```

## Table Extraction

```python
import pdfplumber

class TableExtractor:
    def __init__(self):
        self.client = OpenAI()

    def extract_tables(self, pdf_path: str) -> list[dict]:
        """Extract tables from PDF."""
        tables = []

        with pdfplumber.open(pdf_path) as pdf:
            for page in pdf.pages:
                page_tables = page.extract_tables()
                for table in page_tables:
                    if table:
                        tables.append(self.structure_table(table))

        return tables

    def structure_table(self, raw_table: list) -> dict:
        """Convert raw table to structured format."""
        if not raw_table or not raw_table[0]:
            return {"headers": [], "rows": []}

        # First row as headers
        headers = [h or f"col_{i}" for i, h in enumerate(raw_table[0])]

        # Rest as data rows
        rows = []
        for row in raw_table[1:]:
            row_dict = {}
            for i, cell in enumerate(row):
                if i < len(headers):
                    row_dict[headers[i]] = cell
            rows.append(row_dict)

        return {"headers": headers, "rows": rows}

    def tables_to_json(self, tables: list[dict], context: str = "") -> dict:
        """Use LLM to understand and structure tables."""
        prompt = f"""Analyze these tables and extract structured data.

Tables:
{json.dumps(tables, indent=2)}

{f"Context: {context}" if context else ""}

Return JSON with meaningful field names based on table content."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)
```

## Production Pipeline

```python
from dataclasses import dataclass
from enum import Enum
import asyncio

class DocumentType(Enum):
    INVOICE = "invoice"
    CONTRACT = "contract"
    FORM = "form"
    RECEIPT = "receipt"

@dataclass
class ProcessingResult:
    success: bool
    document_type: DocumentType
    extracted_data: dict
    confidence: float
    errors: list[str]

class ProductionDocPipeline:
    def __init__(self):
        self.vision_processor = VisionDocProcessor()
        self.hybrid_processor = HybridDocProcessor()
        self.table_extractor = TableExtractor()
        self.client = OpenAI()

        self.schemas = {
            DocumentType.INVOICE: {
                "invoice_number": "string",
                "date": "string",
                "vendor": {"name": "string", "address": "string"},
                "total": "number",
                "tax": "number",
                "line_items": [{"description": "string", "amount": "number"}]
            },
            DocumentType.CONTRACT: {
                "title": "string",
                "parties": [{"name": "string", "role": "string"}],
                "effective_date": "string",
                "terms": "string",
                "signatures": [{"name": "string", "date": "string"}]
            },
            DocumentType.RECEIPT: {
                "merchant": "string",
                "date": "string",
                "items": [{"name": "string", "price": "number"}],
                "total": "number",
                "payment_method": "string"
            }
        }

    async def process(self, pdf_path: str) -> ProcessingResult:
        """Process document through full pipeline."""
        try:
            # Step 1: Classify document type
            doc_type = await self.classify_document(pdf_path)

            # Step 2: Get appropriate schema
            schema = self.schemas.get(doc_type, self.schemas[DocumentType.FORM])

            # Step 3: Extract data using best approach
            if doc_type == DocumentType.INVOICE:
                # Use hybrid for better table handling
                data = self.hybrid_processor.process_pdf(pdf_path, schema)
            else:
                # Use vision for general documents
                data = self.vision_processor.process_pdf(pdf_path, schema)

            # Step 4: Extract tables if present
            tables = self.table_extractor.extract_tables(pdf_path)
            if tables:
                data["tables"] = tables

            # Step 5: Validate extraction
            confidence = await self.validate_extraction(data, schema)

            return ProcessingResult(
                success=True,
                document_type=doc_type,
                extracted_data=data,
                confidence=confidence,
                errors=[]
            )

        except Exception as e:
            return ProcessingResult(
                success=False,
                document_type=DocumentType.FORM,
                extracted_data={},
                confidence=0,
                errors=[str(e)]
            )

    async def classify_document(self, pdf_path: str) -> DocumentType:
        """Classify document type using first page."""
        images = convert_from_path(pdf_path, first_page=1, last_page=1)
        buffer = BytesIO()
        images[0].save(buffer, format="PNG")
        img_b64 = base64.b64encode(buffer.getvalue()).decode()

        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": [
                    {"type": "text", "text": "Classify this document. Return only: invoice, contract, receipt, or form"},
                    {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}}
                ]
            }],
            max_tokens=10
        )

        type_str = response.choices[0].message.content.strip().lower()

        try:
            return DocumentType(type_str)
        except ValueError:
            return DocumentType.FORM

    async def validate_extraction(self, data: dict, schema: dict) -> float:
        """Validate extraction quality."""
        # Check completeness
        required_fields = self._get_required_fields(schema)
        filled_fields = sum(1 for f in required_fields if data.get(f))
        completeness = filled_fields / len(required_fields) if required_fields else 1.0

        return completeness

    def _get_required_fields(self, schema: dict) -> list[str]:
        return [k for k in schema.keys() if not k.startswith("_")]
```

## Batch Processing

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class BatchProcessor:
    def __init__(self, max_concurrent: int = 5):
        self.pipeline = ProductionDocPipeline()
        self.max_concurrent = max_concurrent

    async def process_batch(self, pdf_paths: list[str]) -> list[ProcessingResult]:
        """Process multiple documents concurrently."""
        semaphore = asyncio.Semaphore(self.max_concurrent)

        async def process_with_limit(path):
            async with semaphore:
                return await self.pipeline.process(path)

        tasks = [process_with_limit(path) for path in pdf_paths]
        return await asyncio.gather(*tasks)

    def save_results(self, results: list[ProcessingResult], output_dir: str):
        """Save extraction results to JSON files."""
        import os

        os.makedirs(output_dir, exist_ok=True)

        for i, result in enumerate(results):
            output_path = os.path.join(output_dir, f"result_{i}.json")
            with open(output_path, "w") as f:
                json.dump({
                    "success": result.success,
                    "document_type": result.document_type.value,
                    "confidence": result.confidence,
                    "data": result.extracted_data,
                    "errors": result.errors
                }, f, indent=2)

# Usage
async def main():
    processor = BatchProcessor(max_concurrent=3)

    pdf_files = ["doc1.pdf", "doc2.pdf", "doc3.pdf"]
    results = await processor.process_batch(pdf_files)

    processor.save_results(results, "output/")

asyncio.run(main())
```

## Cost Optimization

```python
# Gemini Flash for cost-effective OCR ($1 per 6000 pages)
import google.generativeai as genai

class CostOptimizedProcessor:
    def __init__(self):
        genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
        self.model = genai.GenerativeModel("gemini-2.0-flash")

    def process_pdf(self, pdf_path: str, schema: dict) -> dict:
        """Process with Gemini Flash for cost efficiency."""
        # Upload PDF
        pdf_file = genai.upload_file(pdf_path)

        prompt = f"""Extract data from this document:
{json.dumps(schema, indent=2)}
Return JSON only."""

        response = self.model.generate_content([pdf_file, prompt])
        return json.loads(response.text)
```

## Resources

- [LlamaIndex PDF Guide](https://www.llamaindex.ai/blog/beyond-ocr-how-llms-are-revolutionizing-pdf-parsing)
- [Google Document AI](https://cloud.google.com/document-ai)
- [Mistral Document AI](https://mistral.ai/solutions/document-ai)
- [pdfplumber](https://github.com/jsvine/pdfplumber)
- [EasyOCR](https://github.com/JaidedAI/EasyOCR)

---

*Questions about document processing pipelines? [Let me know](mailto:jordan@jordananderson.us).*
