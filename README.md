# Affinda Skills

A collection of agent skills for working with Affinda's document AI platform.

## Installation

```bash
# Install all skills
npx skills add affinda/affinda-skills

# Install a specific skill
npx skills add affinda/affinda-skills@resume-parsing
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `resume-parsing` | Best practices for parsing and extracting data from resumes using Affinda's API |
| `document-classification` | Guidelines for classifying documents into categories using Affinda |
| `invoice-extraction` | Rules and patterns for extracting structured data from invoices |

## Publishing

Skills follow the [Agent Skills](https://skills.sh) format. Each skill lives in `skills/<name>/SKILL.md`.
