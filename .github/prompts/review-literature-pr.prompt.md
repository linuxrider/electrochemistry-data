# Skill: Review Literature Addition PR

## Purpose

Automate the review of pull requests that add new literature data to the echemdb electrochemistry-data repository. This replaces the manual review checklist from `.github/pull_request_template.md`.

## When to Use

Use this skill when:
- A PR adds new files under `literature/svgdigitizer/` or `literature/source_data/`
- The user asks to "review" a literature PR or data submission
- Validation errors occur and the user wants a comprehensive review

## Review Process

### Step 1: Identify Changed Entries

Run `git diff main --name-only` to find new/modified files. Focus on directories under:
- `literature/svgdigitizer/{identifier}/` — SVG-digitized data
- `literature/source_data/{identifier}/` — Raw experimental data
- `literature/bibliography/bibliography.bib` — Bibliography entries

### Step 2: Run Automated Validation

```bash
pixi run -e dev validate-input
```

This runs schema validation, filename checks, identifier matching, and bib key validation. Report any failures.

### Step 3: Run the Review Module

```python
from echemdb_ecdata.review import review_entry, format_report
report = review_entry("literature/svgdigitizer/{identifier}")
print(format_report(report))
```

This performs the following checks automatically:

#### Filename Checks
- All filenames and directory names are lowercase
- Filenames follow `{citationKey}_f{figure}_{curve}.{ext}` pattern
- Each SVG has a matching YAML file
- Filenames start with the directory name (= citation key)

#### Bibliography Checks
- `citationKey` in YAML matches the directory name
- Bibliography entry exists in `bibliography.bib`
- Computed identifier (via `svgdigitizer.pdf.Pdf.build_identifier`) matches the citation key
- Title has no spurious spaces before parentheses (e.g., `Pt (111)` → `Pt(111)`)

#### SVG Checks
- Has `tags` text label (BCV, ORR, etc.)
- Has `figure` text label in correct format (number only, not prefixed with 'f')
- Has `curve` text label (short, unambiguous)
- Has `scan rate` text label
- All axis labels present (E1, E2, j1, j2)
- Reference electrode in SVG axes matches YAML

#### YAML Checks
- `citationKey` matches directory name
- Required sections present: `curation`, `source`, `system`
- Source URL is a valid DOI
- Electrolyte has `type` and `components`
- Working electrode and reference electrode defined
- Curator has ORCID

#### PDF Cross-Validation
The review module downloads the paper via DOI and extracts text to verify:
- Electrolyte components mentioned in paper
- Concentration values match
- Electrode material matches
- Crystallographic orientation matches
- pH value matches
- Scan rate matches
- Reference electrode matches
- Purity/grade info matches
- Supplier names match

### Step 4: Manual Cross-Checks (Agent Should Verify)

After automated checks, the agent should additionally verify by reading the PDF text:

1. **Figure identification**: Confirm the figure number in the SVG/YAML matches the actual figure in the paper
2. **Axis units**: Confirm the axis units in the SVG match those in the paper's figure
3. **Experimental conditions**: Spot-check that temperature, pH, and concentrations are correctly transcribed
4. **Electrode preparation**: Verify preparation procedure description matches the paper
5. **Comments**: Verify any comments are complete sentences ending with a period

### Step 5: Report Findings

Format the review as:

```markdown
## Literature Review: {identifier}

### Automated Validation
- [ ] `validate-input` passes
- [ ] Schema validation passes

### Filename Checks
- [x] All lowercase
- [x] Pattern matches
...

### PDF Cross-Validation
- [x] Electrolyte components verified
- [x] Concentrations match
...

### Manual Verification Notes
- {Any discrepancies found by reading the PDF}

### Summary
{errors} errors, {warnings} warnings
{Recommended actions}
```

## Key References

- PR checklist: `.github/pull_request_template.md`
- Naming convention: `{citationKey}_f{figure}_{curve}.{ext}` where citationKey = `{author}_{year}_{title_word}_{first_page}`
- Review module: `echemdb_ecdata/review.py`
- Validation: `pixi run -e dev validate-input`
- Bibliography: `literature/bibliography/bibliography.bib`
- Metadata schema: https://github.com/echemdb/metadata-schema

## Common Issues

1. **BIB MISMATCH**: Citation key in YAML doesn't match bibliography.bib — check spelling and that the bib entry was added
2. **Figure label format**: Should be `2` not `f2` in the SVG text label
3. **Missing tags**: Every SVG must have a `tags:` text node
4. **Uppercase in filenames**: All identifiers must be lowercase (Windows compatibility)
5. **Missing counter electrode**: The PR checklist expects WE, RE, and optionally CE
