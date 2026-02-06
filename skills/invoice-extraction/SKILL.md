---
name: invoice-extraction
description: Rules and patterns for extracting structured data from invoices using Affinda's document AI. Use when building accounts payable automation, invoice processing pipelines, or financial document workflows.
---

# Invoice Extraction

This skill provides best practices for extracting structured data from invoices using Affinda's Invoice Parser API.

## When to Use This Skill

Use this skill when the user:

- Is building an accounts payable automation system
- Needs to extract line items, totals, dates, and vendor info from invoices
- Is integrating invoice parsing with accounting software (Xero, QuickBooks, etc.)
- Wants to handle invoices in multiple currencies or languages
- Needs to validate extracted invoice data against business rules

## Key Principles

### 1. Extract All Key Fields

Invoices typically contain these extractable fields:

| Field | Description |
|-------|-------------|
| `invoice_number` | Unique invoice identifier |
| `invoice_date` | Date the invoice was issued |
| `due_date` | Payment due date |
| `supplier` | Vendor/supplier details |
| `customer` | Buyer/customer details |
| `line_items` | Individual items with descriptions, quantities, and prices |
| `subtotal` | Pre-tax total |
| `tax` | Tax amount |
| `total` | Final amount due |
| `currency` | Currency code |
| `payment_details` | Bank account or payment info |

### 2. Validate Line Item Totals

Always cross-check that line item amounts sum to the subtotal, and that subtotal + tax equals the total.

```python
line_item_sum = sum(item.total for item in document.data.line_items if item.total)
expected_subtotal = document.data.subtotal.value if document.data.subtotal else None

if expected_subtotal and abs(line_item_sum - expected_subtotal) > 0.01:
    flag_for_review(document, "Line items don't sum to subtotal")
```

### 3. Handle Currency and Locale

Invoices may use different decimal separators (comma vs period) and currency formats. Rely on Affinda's normalized output rather than parsing raw text.

### 4. Duplicate Detection

Implement duplicate invoice detection using invoice number + supplier + amount + date as a composite key to prevent double payments.

### 5. Table Extraction Quality

Line item extraction accuracy depends on table structure. For complex or nested tables, verify extraction results and consider using custom extraction fields.

## Common Pitfalls

- Not handling credit notes (negative invoices) in the pipeline
- Assuming all invoices have line items (some only have a total)
- Ignoring multi-currency scenarios in aggregation
- Not validating tax calculations against expected tax rates
- Failing to handle multi-page invoices where tables span pages
