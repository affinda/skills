---
name: document-classification
description: Guidelines for classifying documents into categories using Affinda's document AI. Use when building document sorting workflows, automating mailroom processing, or routing documents by type.
---

# Document Classification

This skill provides best practices for using Affinda's API to automatically classify documents into predefined or custom categories.

## When to Use This Skill

Use this skill when the user:

- Needs to sort incoming documents by type (invoice, receipt, contract, resume, etc.)
- Is building an automated mailroom or document intake system
- Wants to route documents to different processing pipelines based on type
- Needs to create custom document classification categories

## Key Principles

### 1. Use Workspaces and Collections

Set up workspaces with the appropriate document types enabled. Each collection inherits the workspace configuration.

### 2. Leverage Built-in Document Types

Affinda supports many document types out of the box:

- Resumes / CVs
- Invoices
- Receipts
- Bank Statements
- Tax Forms
- Contracts
- Letters

Use these before creating custom classifiers.

### 3. Handle Low-Confidence Classifications

Always check the confidence score and implement fallback logic for uncertain classifications.

```python
classification = document.meta.document_type
confidence = document.meta.confidence

if confidence < 0.7:
    # Route to manual review queue
    send_to_review(document)
else:
    # Process automatically
    process_document(document, classification)
```

### 4. Batch Processing

When classifying large volumes, use batch endpoints and implement proper rate limiting and error handling.

### 5. Multi-Page Documents

Some documents span multiple pages. Ensure your pipeline handles page grouping correctly rather than classifying each page independently.

## Common Pitfalls

- Classifying scanned documents without OCR preprocessing
- Not handling mixed-type document bundles (e.g., a PDF with an invoice and a receipt)
- Ignoring the distinction between document type and document subtype
- Setting confidence thresholds too low, leading to misrouted documents
