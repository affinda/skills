---
name: resume-parsing
description: Best practices for parsing and extracting structured data from resumes and CVs using Affinda's document AI API. Use when building resume parsing workflows, integrating with ATS systems, or extracting candidate information.
---

# Resume Parsing Best Practices

This skill provides guidelines for working with Affinda's Resume Parser API to extract structured candidate data from resumes and CVs.

## When to Use This Skill

Use this skill when the user:

- Is building a resume/CV parsing pipeline
- Wants to extract candidate information (name, contact, experience, education, skills)
- Is integrating Affinda's resume parser into an ATS or HR system
- Needs to handle multi-page or multi-format resumes (PDF, DOCX, images)
- Wants to normalize parsed resume data

## Key Principles

### 1. Use the Right Document Type

Always specify `resume` as the document type when creating a document for parsing. This ensures the correct extraction model is used.

### 2. Handle Asynchronous Processing

Resume parsing is asynchronous. Always poll for completion or use webhooks rather than assuming instant results.

```python
# Good: Use wait=True or poll for status
document = client.create_document(file=resume_file, collection=collection_id, wait=True)

# Bad: Assuming document is ready immediately after creation
document = client.create_document(file=resume_file, collection=collection_id)
print(document.data)  # May be None if still processing
```

### 3. Validate Extracted Fields

Not all resumes contain all fields. Always check for `None` before accessing nested data.

```python
# Good: Safe access with fallbacks
name = document.data.name.raw if document.data.name else "Unknown"
email = document.data.emails[0] if document.data.emails else None

# Bad: Assuming fields exist
name = document.data.name.raw  # May raise AttributeError
```

### 4. Handle Multiple Work Experiences

Resumes often have multiple work experience entries. Iterate and validate each one.

### 5. Use Collections for Organization

Group related documents into collections (e.g., by job posting, department, or recruitment campaign) for easier management and retrieval.

## Common Pitfalls

- Not handling password-protected PDFs gracefully
- Ignoring confidence scores on extracted fields
- Not normalizing dates across different resume formats
- Forgetting to clean up temporary files after upload
- Not handling rate limits on high-volume parsing
