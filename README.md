# Feature Specifications

This directory contains all feature requirement specs for the BBS application.

## Structure

```
features/
├── README.md                  # This file (index)
├── _template.md               # Spec template
├── _edgecases-template.md     # Edge cases template
├── {feature-slug}/
│   ├── spec.md                # Feature specification (required)
│   ├── edgecases.md           # Edge cases & decisions (required)
│   ├── api-contract.md        # API endpoint details (optional)
│   ├── schema.md              # DB schema changes (optional)
│   └── notes.md               # Open questions, decisions (optional)
```

## Naming Convention

- Folder name: lowercase kebab-case slug (e.g., `gpa-threshold-config`, `student-attendance-export`)
- Spec file: always `spec.md`
- Edge cases file: always `edgecases.md`

## Required Files

Every feature spec **must** have:
1. `spec.md` — Main specification (what, why, acceptance criteria)
2. `edgecases.md` — Edge cases with options and decisions

Optional files (create if needed):
- `api-contract.md` — API endpoint details
- `schema.md` — Database schema changes
- `notes.md` — Open questions, decisions, discussion notes

## Existing Specs

| Feature | Status | Spec Path |
|---------|--------|-----------|
| GPA Threshold Config Fix | Draft | `gpa-threshold-config/spec.md` |
| External Attendance Integration | Draft | `external-attendance-integration/spec.md` |

---

## How to Use

### For PM / Product Owner
1. Create folder: `features/{feature-slug}/`
2. Copy `_template.md` → `spec.md` and fill in
3. Copy `_edgecases-template.md` → `edgecases.md` and list potential edge cases
4. Mark as `Status: Draft` in frontmatter

### For Engineer
1. Read `spec.md` for the feature
2. Read `edgecases.md` for edge case decisions
3. Reference `api-contract.md` and `schema.md` if they exist
4. Check `notes.md` for open questions / decisions

### For AI Agent (Pi)
- Load `.pi/context/codebase-overview.md` for codebase structure
- Load `features/{feature}/spec.md` as primary context
- Load `features/{feature}/edgecases.md` for edge case handling
- Supplement with `api-contract.md` / `schema.md` as needed

## Workflow

1. **Draft** — PM writes spec + edge cases
2. **Review** — Team reviews, discuss edge cases
3. **Decide** — Fill in edge case decisions
4. **Implement** — Engineer builds based on spec + edge case decisions
5. **Done** — Update status to `implemented`
