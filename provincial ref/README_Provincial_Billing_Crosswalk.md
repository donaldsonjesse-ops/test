# Provincial Physician Billing Crosswalk System

## Technical Specification & Implementation Guide

**Version:** 1.0  
**Date:** January 2026  
**Purpose:** Benchmarking Alberta physician billing codes against other Canadian provinces

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Solution Architecture](#3-solution-architecture)
4. [Data Schema](#4-data-schema)
5. [Phase 1: Reference Extraction](#5-phase-1-reference-extraction)
6. [Phase 2: Code Matching](#6-phase-2-code-matching)
7. [Phase 3: Ontology Population](#7-phase-3-ontology-population)
8. [Implementation Details](#8-implementation-details)
9. [File Structure](#9-file-structure)
10. [Quality Assurance](#10-quality-assurance)

---

## 1. Executive Summary

### What This System Does

This system enables comparison of Alberta physician billing codes against equivalent codes in Ontario, Saskatchewan, Manitoba, and British Columbia. It answers questions like: "If Alberta pays $25.09 for a telehealth consultation (03.03CV), what do other provinces pay for the same clinical service?"

### Why It's Complex

Provincial billing codes are not directly comparable because:

- **Different code structures**: Alberta uses XX.XXYZ format; Ontario uses letter-number combinations; Manitoba uses 4-digit tariff numbers
- **Different bundling**: One province may have a single code where another splits into base + add-ons
- **Different modifiers**: Time premiums, age premiums, location premiums vary by province
- **Inherited rules**: Most billing rules are stated in preambles, not per-code

### The Three-Phase Approach

| Phase | What It Does | When It Runs | Output |
|-------|--------------|--------------|--------|
| **Phase 1: Reference Extraction** | Builds complete reference database for each province | Once per province per year (when fee schedules update) | `{province}_reference.json` |
| **Phase 2: Code Matching** | Finds equivalent codes across provinces for a given Alberta code | As needed, per Alberta code | Match results with reasoning |
| **Phase 3: Ontology Population** | Enriches matched codes with comparable attributes | After matching | Final crosswalk output |

### Key Insight

**Build the reference first, match second, populate third.** This inverts the naive approach (match then extract) and dramatically improves accuracy while reducing per-query cost.

---

## 2. Problem Statement

### Current State

The existing code (`crosswalk_telehealth_all_provincesV3.py`) successfully matches Alberta codes to provincial equivalents. It processes PDFs in 10-page batches, asks an LLM to identify matching codes, and outputs results to Excel.

**What works well:**
- Code matching logic is sound
- Deduplication handles multiple occurrences
- Section tracking provides context
- Output structure is clean

**What's missing:**
- No extraction of billing rules, exclusions, time requirements
- No capture of modifier/premium codes that affect total payment
- No systematic handling of inherited rules from preambles
- No reference database for reuse across multiple Alberta codes

### The Boss's Request

A comprehensive taxonomy capturing: modality, delivery setting, age premiums, maximums, exclusions, and other cost-affecting variants. The Universal Billing Code Template attempts this with ~474 fields across 25+ categories.

### Why the Universal Template Fails

1. **Too granular**: Most fields will be empty or "not specified" for most codes
2. **AI hallucination risk**: When asked to fill 474 fields, AI invents plausible-sounding but incorrect values
3. **Information isn't structured that way**: Fee schedules state exceptions, not comprehensive attributes
4. **No inheritance model**: Codes inherit rules from sections and preambles; per-code extraction misses this

### The Solution

A **simplified ontology** (~30 fields) focused on attributes that:
- Actually affect cost comparisons
- Are systematically extractable from fee schedules
- Can be traced to specific source pages
- Handle inheritance explicitly

---

## 3. Solution Architecture

### System Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PHASE 1: REFERENCE EXTRACTION               │
│                         (Once per province per year)                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Provincial PDF ──► Pass 1: Structure Mapping                      │
│        │                    │                                       │
│        │                    ▼                                       │
│        │            document_map.json                               │
│        │                    │                                       │
│        ├──────────► Pass 2: Preamble Extraction                     │
│        │                    │                                       │
│        │                    ▼                                       │
│        │            global_rules.json                               │
│        │                    │                                       │
│        ├──────────► Pass 3: Section Rules                           │
│        │                    │                                       │
│        │                    ▼                                       │
│        │            section_rules.json                              │
│        │                    │                                       │
│        ├──────────► Pass 4: Code Extraction                         │
│        │                    │                                       │
│        │                    ▼                                       │
│        │            codes_raw.json                                  │
│        │                    │                                       │
│        └──────────► Pass 5-7: Relationships, Inference, Validation  │
│                             │                                       │
│                             ▼                                       │
│                    {province}_reference.json                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         PHASE 2: CODE MATCHING                      │
│                         (Per Alberta code)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Alberta Code Definition                                           │
│        +                                                            │
│   {province}_reference.json ──► LLM Matching ──► matched_codes.json │
│                                                                     │
│   (Uses existing matching logic, but now has full reference)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      PHASE 3: ONTOLOGY POPULATION                   │
│                      (Post-match enrichment)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   matched_codes.json                                                │
│        +                                                            │
│   {province}_reference.json ──► Lookup & Synthesis ──► crosswalk.json
│                                                                     │
│   (Deterministic lookup, minimal AI inference)                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Why This Order

| Order | Benefit |
|-------|---------|
| Reference first | Built once, reused for every Alberta code |
| Match second | Matching uses structured reference, not raw PDF |
| Populate third | Ontology fields are lookups, not extractions |

---

## 4. Data Schema

### 4.1 Province Reference Database

The reference database for each province contains seven interconnected tables.

#### 4.1.1 Metadata

| Field | Type | Description |
|-------|------|-------------|
| province_code | string | AB, ON, SK, MB, BC |
| province_name | string | Full name |
| document_name | string | Source PDF filename |
| document_version | string | e.g., "April 1, 2024" |
| effective_date | date | When schedule took effect |
| extraction_date | date | When reference was built |
| total_pages | integer | PDF page count |
| extractor_version | string | Version of extraction code |

#### 4.1.2 Global Rules

| Field | Type | Description |
|-------|------|-------------|
| rule_id | string | Rule identifier (e.g., "Rule 7", "Preamble 3.2") |
| rule_title | string | Title if present |
| rule_text | text | Full verbatim text |
| applies_to_sections | array[string] | Section codes this applies to, or ["ALL"] |
| applies_to_codes | array[string] | Specific codes if enumerated |
| rule_type | enum | general, billing, exclusion, time, documentation, eligibility |
| source_pages | array[integer] | Page numbers where rule appears |

**rule_type enum values:**
- `general` - Definitions, general guidance
- `billing` - How to submit claims
- `exclusion` - What cannot be billed together
- `time` - Time requirements, documentation
- `documentation` - Record-keeping requirements
- `eligibility` - Who can bill, patient eligibility

#### 4.1.3 Sections

| Field | Type | Description |
|-------|------|-------------|
| section_code | string | e.g., "01", "02", "13-4" |
| section_name | string | e.g., "Internal Medicine", "Paediatrics" |
| page_start | integer | First page of section |
| page_end | integer | Last page of section |
| section_preamble | text | Section-specific rules text |
| parent_section | string | If subsection, reference to parent |
| specialty_type | enum | gp, specialist, both |
| hospital_premium_applicable | boolean | Whether hospital premium applies |
| hospital_premium_percentage | decimal | Premium percentage if applicable |
| hospital_premium_codes | array[string] | Specific codes if enumerated |

#### 4.1.4 Codes

| Field | Type | Description |
|-------|------|-------------|
| code | string | Billing code (e.g., "8340", "A101") |
| code_variant | string | Suffix if applicable (e.g., "A") |
| description | string | Short official description |
| description_full | text | Full text including all notes |
| base_fee | decimal | Base rate in dollars |
| fee_unit | enum | flat, per_unit, per_15min, percentage |
| fee_basis | string | If percentage, what it's percentage of |
| section_code | string | FK to sections |
| source_page | integer | Primary page where code is defined |
| all_pages | array[integer] | All pages where code appears |
| notes_verbatim | text | All note text exactly as written |
| status | enum | active, deleted, amended |
| status_date | date | If deleted/amended, effective date |

**fee_unit enum values:**
- `flat` - Fixed dollar amount
- `per_unit` - Per unit (e.g., per 15 minutes)
- `per_15min` - Specifically per 15-minute increment
- `percentage` - Percentage of another fee

#### 4.1.5 Code Attributes

| Field | Type | Description |
|-------|------|-------------|
| code | string | FK to codes |
| attribute_type | enum | See attribute_type enum below |
| attribute_value | string | The extracted value |
| attribute_source | enum | explicit, inherited_section, inherited_global, inferred |
| source_page | integer | Page where stated (if explicit) |
| source_rule | string | Rule ID (if inherited) |
| confidence | enum | high, medium, low |
| extraction_notes | text | Notes on how value was determined |

**attribute_type enum values:**

| Category | Attribute Types |
|----------|-----------------|
| Modality | modality_telephone, modality_video, modality_both, modality_in_person, modality_asynchronous |
| Time | minimum_time_minutes, maximum_time_minutes, time_documentation_required |
| Initiation | patient_initiated_required, physician_initiated_allowed |
| Referral | referral_required, referral_not_required, self_referral_allowed |
| Authorization | certification_required, approval_required, written_report_required |
| Frequency | frequency_per_day, frequency_per_week, frequency_per_month, frequency_per_year, frequency_per_lifetime |
| Restrictions | age_restriction_min, age_restriction_max, setting_restriction, provider_type_restriction |

**attribute_source enum values:**
- `explicit` - Stated directly in code notes
- `inherited_section` - Inherited from section preamble
- `inherited_global` - Inherited from global rules
- `inferred` - Derived from description/context

#### 4.1.6 Modifiers

| Field | Type | Description |
|-------|------|-------------|
| modifier_code | string | Code for the modifier (e.g., "5530") |
| modifier_name | string | Descriptive name |
| modifier_type | enum | time_premium, age_premium, location_premium, complexity, setting, add_on |
| modifier_value | decimal | Amount or percentage |
| modifier_unit | enum | flat, percentage |
| applicable_to_codes | array[string] | Specific codes if listed |
| applicable_to_sections | array[string] | Sections if broad applicability |
| exclusion_codes | array[string] | Codes it cannot be used with |
| conditions_verbatim | text | Full conditions text |
| source_page | integer | Where defined |

**modifier_type enum values:**
- `time_premium` - After hours, weekend, holiday
- `age_premium` - Pediatric, geriatric
- `location_premium` - Rural, remote, northern
- `complexity` - Extended visit, comprehensive
- `setting` - Hospital, facility
- `add_on` - Additional service component

#### 4.1.7 Exclusions

| Field | Type | Description |
|-------|------|-------------|
| exclusion_id | string | Generated unique ID |
| exclusion_type | enum | same_day, same_provider, same_patient, same_encounter, mutually_exclusive |
| code_a | string | First code in exclusion |
| code_b | string | Second code (if pairwise) |
| code_list | array[string] | All codes (if group exclusion) |
| direction | enum | bidirectional, a_excludes_b, b_excludes_a |
| conditions | text | Any conditions on the exclusion |
| source_page | integer | Where stated |
| source_rule | string | Rule ID if from preamble |

#### 4.1.8 Relationships

| Field | Type | Description |
|-------|------|-------------|
| relationship_id | string | Generated unique ID |
| relationship_type | enum | add_on_to, variant_of, replaces, see_also, same_service_different_setting |
| code_from | string | Source code |
| code_to | string | Target code |
| conditions | text | When relationship applies |
| source_page | integer | Where stated |

---

### 4.2 Crosswalk Output Schema

The final output after matching and ontology population.

#### 4.2.1 Alberta Source Record

| Field | Type | Description |
|-------|------|-------------|
| ab_code | string | Alberta billing code |
| ab_description | string | Description |
| ab_fee | decimal | Base fee |
| ab_clinical_definition | text | Full clinical service definition |
| ab_modality | string | telephone, video, both |
| ab_minimum_time | integer | Minimum minutes required |
| ab_exclusions | array[string] | Same-day exclusion codes |
| ab_patient_initiated | boolean | Must be patient-initiated |
| ab_documentation_required | text | Documentation requirements |

#### 4.2.2 Matched Code Record

| Field | Type | Description |
|-------|------|-------------|
| province | string | Province code |
| code | string | Matched billing code |
| match_type | enum | primary, add_on, alternative |
| description | string | From reference |
| base_fee | decimal | From reference |
| modality | string | From reference |
| match_confidence | enum | exact, high, moderate, partial |
| match_reasoning | text | Why this code matches |
| source_page | integer | Primary page in fee schedule |

#### 4.2.3 Comparison Attributes

| Field | Type | Description |
|-------|------|-------------|
| fee_ratio | decimal | target_fee / ab_fee |
| fee_difference | decimal | target_fee - ab_fee |
| modality_match | boolean | Same modality coverage |
| time_requirement_comparison | enum | same, target_more_restrictive, target_less_restrictive, not_comparable |
| exclusions_similar | boolean | Similar exclusion structure |
| key_differences | array[string] | Human-readable list of differences |
| comparability_score | integer | 1-100 overall comparability |
| comparability_notes | text | Explanation of limitations |

#### 4.2.4 Applicable Modifiers

| Field | Type | Description |
|-------|------|-------------|
| modifier_code | string | Modifier code |
| modifier_name | string | Description |
| modifier_value | decimal | Amount or percentage |
| modifier_type | string | Type of modifier |
| conditions | text | When applicable |

#### 4.2.5 Provenance

| Field | Type | Description |
|-------|------|-------------|
| reference_version | string | Version of province reference used |
| reference_date | date | When reference was extracted |
| match_date | date | When matching was performed |
| match_model | string | AI model used for matching |
| all_source_pages | array[integer] | All pages referenced |
| rule_references | array[string] | Rules that informed attributes |
| requires_manual_review | boolean | Flag for QA |
| review_reasons | array[string] | Why review is needed |

---

## 5. Phase 1: Reference Extraction

### Overview

Reference extraction processes a provincial fee schedule PDF once to create a structured database. This database is then reused for all subsequent matching operations against that province.

### Estimated Effort

| Province | Typical Pages | Estimated Time | LLM Calls |
|----------|---------------|----------------|-----------|
| Ontario | 800-900 | 60-90 min | 100-150 |
| Manitoba | 500-550 | 45-70 min | 70-100 |
| Saskatchewan | 400-450 | 40-60 min | 60-90 |
| British Columbia | 600-700 | 50-80 min | 80-120 |

**Total one-time cost: ~4-6 hours for all provinces**

### Pass 1: Document Structure Mapping

**Purpose:** Create a map of the document's structure without heavy LLM usage.

**Input:** Raw PDF

**Process:**
```
For each page 1 to N:
    1. Extract raw text using pdfplumber
    2. Apply heuristic detection:
       - TOC pages: Look for "Table of Contents", page number patterns like "... 42"
       - Preamble pages: Look for "Rule X—", "Introduction", "General Information"
       - Section headers: Look for ALL CAPS lines, patterns like "(01)", "(02)"
       - Code pages: Look for fee patterns like "$XX.XX", tariff number patterns
    3. Record: page_number, detected_type, detected_section, header_text
```

**Heuristics for page type detection:**

| Page Type | Detection Patterns |
|-----------|-------------------|
| TOC | "Table of Contents", "INDEX", lines with "...." followed by numbers |
| Preamble | "Rule \d+", "Introduction", "General", pages 1-50 typically |
| Section Header | ALL CAPS line > 20 chars, followed by "(XX)" pattern |
| Code Listing | Multiple lines matching `^\d{4}\s+\w+.*\$\d+\.\d{2}` or similar |
| General Schedule | "General Schedule", "General Preamble" |

**Output:** `document_map.json`

```json
{
  "province": "MB",
  "total_pages": 537,
  "toc_pages": [2, 3, 4],
  "preamble_pages": [5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15],
  "general_schedule_pages": [48, 49, 50, 51, 52, 53, 54, 55],
  "sections": [
    {
      "section_code": "01",
      "section_name": "Internal Medicine",
      "page_start": 83,
      "page_end": 142,
      "detected_subsections": ["Office, Home Visits", "Virtual Visits", "Hospital Care"]
    },
    {
      "section_code": "02", 
      "section_name": "Paediatrics",
      "page_start": 143,
      "page_end": 198,
      "detected_subsections": ["Office, Home Visits", "Consultations"]
    }
  ]
}
```

### Pass 2: Global Rules Extraction

**Purpose:** Extract all rules from preamble and general schedule sections.

**Input:** Pages identified as preamble or general schedule in document_map

**Process:**
```
For each batch of 5-10 preamble pages:
    1. Concatenate page text
    2. Send to LLM with prompt (see below)
    3. Parse response into rule records
    4. Append to global_rules collection
```

**LLM Prompt:**

```
You are extracting billing rules from a provincial physician fee schedule.

DOCUMENT: {province_name} Physician Payment Schedule
PAGES: {page_start} to {page_end}

TEXT:
{page_text}

TASK:
Extract every rule, regulation, or policy statement. For each, provide:

1. rule_id: The rule identifier (e.g., "Rule 7", "Preamble 3.2", "General Policy 4")
2. rule_title: Title if present
3. rule_text: Complete verbatim text of the rule
4. rule_type: One of: general, billing, exclusion, time, documentation, eligibility
5. applies_to_sections: List of section codes this applies to, or ["ALL"]
6. applies_to_codes: List of specific billing codes mentioned, or []
7. source_page: Page number where rule appears

Return JSON array. Include EVERY rule, even if lengthy.
Do not summarize rule_text - include complete verbatim text.

JSON:
```

**Output:** `global_rules.json`

```json
[
  {
    "rule_id": "Rule 7",
    "rule_title": "Consultations",
    "rule_text": "A consultation is a service rendered by a specialist...[full text]",
    "rule_type": "general",
    "applies_to_sections": ["ALL"],
    "applies_to_codes": ["8550", "8535"],
    "source_page": 12
  }
]
```

### Pass 3: Section Rules Extraction

**Purpose:** Extract section-specific preambles and rules.

**Input:** First 2-3 pages of each section from document_map

**Process:**
```
For each section in document_map.sections:
    1. Extract text from first 2-3 pages
    2. Send to LLM with section context
    3. Parse response into section record
```

**LLM Prompt:**

```
You are extracting section-specific rules from a physician fee schedule.

PROVINCE: {province_name}
SECTION: {section_code} - {section_name}
PAGES: {page_start} to {page_start + 2}

TEXT:
{section_preamble_text}

TASK:
Extract section-specific information:

1. section_preamble: Full text of any section preamble/introduction
2. hospital_premium_applicable: true/false - does a hospital premium apply to codes in this section?
3. hospital_premium_percentage: If applicable, what percentage?
4. hospital_premium_codes: If specific codes listed, include them; otherwise []
5. section_specific_rules: Array of any rules specific to this section
6. specialty_type: Is this for "gp", "specialist", or "both"?
7. referenced_global_rules: Which global rules are referenced? (e.g., "See Rules 7 to 10")

Return JSON object.

JSON:
```

**Output:** `section_rules.json`

```json
[
  {
    "section_code": "01",
    "section_name": "Internal Medicine",
    "section_preamble": "These benefits cannot be correctly interpreted without reference to the Rules of Application.",
    "hospital_premium_applicable": true,
    "hospital_premium_percentage": 15,
    "hospital_premium_codes": ["8300", "8301", "8302", "..."],
    "section_specific_rules": [],
    "specialty_type": "specialist",
    "referenced_global_rules": ["Rules of Application"],
    "source_pages": [83, 84]
  }
]
```

### Pass 4: Code Extraction

**Purpose:** Extract every billing code with full details.

**Input:** All code listing pages (typically 80% of document)

**Process:**

Two strategies depending on document size:

**Strategy A: Page-by-page (for documents > 600 pages)**
```
For each page in code_pages:
    1. Get section context from document_map
    2. Get relevant section rules
    3. Send page + context to LLM
    4. Parse and append codes
```

**Strategy B: Section batching (for documents < 600 pages)**
```
For each section:
    1. Concatenate all section pages
    2. If > 30 pages, split into chunks with 2-page overlap
    3. Send section chunk + section rules to LLM
    4. Parse and append codes
    5. Deduplicate across chunks
```

**LLM Prompt:**

```
You are extracting billing codes from a physician fee schedule.

PROVINCE: {province_name}
SECTION: {section_code} - {section_name}
PAGES: {page_range}

SECTION CONTEXT:
{section_preamble}
Hospital Premium: {hospital_premium_info}

TEXT:
{page_text}

TASK:
Extract EVERY billing code on these pages. For each code:

1. code: The billing code (e.g., "8340", "A101")
2. code_variant: Any suffix/variant indicator
3. description: Short description
4. description_full: Complete description including all text
5. base_fee: Dollar amount (number only, no $)
6. fee_unit: One of: flat, per_unit, per_15min, percentage
7. fee_basis: If percentage, what is it a percentage of?
8. notes_verbatim: ALL note text exactly as written (preserve formatting)
9. source_page: Page number

CRITICAL:
- Extract EVERY code, even if similar to others
- Include COMPLETE notes - do not summarize
- If fee is "By Report" or variable, set base_fee to null
- Preserve exact wording of notes

Return JSON array of codes.

JSON:
```

**Output:** `codes_raw.json`

```json
[
  {
    "code": "8340",
    "code_variant": null,
    "description": "Episodic virtual visit by phone",
    "description_full": "Episodic virtual visit by phone",
    "base_fee": 20.40,
    "fee_unit": "flat",
    "fee_basis": null,
    "notes_verbatim": null,
    "source_page": 85,
    "section_code": "01"
  },
  {
    "code": "8321",
    "code_variant": null,
    "description": "Virtual visit by telephone or video",
    "description_full": "Virtual visit by telephone or video",
    "base_fee": 59.05,
    "fee_unit": "flat",
    "fee_basis": null,
    "notes_verbatim": null,
    "source_page": 85,
    "section_code": "01"
  }
]
```

### Pass 5: Relationship Extraction

**Purpose:** Parse code notes to extract exclusions, add-on relationships, and cross-references.

**Input:** `codes_raw.json`

**Process:**
```
For each code in codes_raw:
    1. Scan notes_verbatim and description_full for patterns
    2. Apply regex patterns (see below)
    3. For ambiguous cases, use LLM clarification
    4. Create exclusion and relationship records
```

**Regex Patterns:**

| Pattern | Meaning | Example |
|---------|---------|---------|
| `cannot be (claimed\|billed) with` | Exclusion | "cannot be claimed with 8321" |
| `not payable (with\|same day as)` | Same-day exclusion | "not payable same day as 03.01AD" |
| `add(ed)? to` | Add-on relationship | "add to visit fee" |
| `in addition to` | Can be billed together | "may be claimed in addition to 8540" |
| `see (Rule\|Tariff)` | Cross-reference | "See Rule 7" |
| `maximum of (\d+) per (day\|week\|month\|year)` | Frequency limit | "maximum of 4 per 12-month period" |
| `replaces` | Replacement | "replaces tariff 8399" |
| `minimum of (\d+) minutes` | Time requirement | "minimum of 30 minutes" |

**Output:** `exclusions.json` and `relationships.json`

### Pass 6: Attribute Inference

**Purpose:** Populate code attributes, handling inheritance.

**Input:** `codes_raw.json`, `global_rules.json`, `section_rules.json`

**Process:**
```
For each code:
    1. Check for explicit attributes in notes_verbatim
    2. Check section rules for inherited attributes
    3. Check global rules for inherited attributes
    4. Apply inference rules for remaining attributes
    5. Record source and confidence for each attribute
```

**Inference Rules:**

| Attribute | Inference Logic |
|-----------|-----------------|
| modality | If description contains "telephone" → telephone; "video" → video; "telephone or video" → both |
| minimum_time | Check for "minimum of X minutes" in notes; check section default |
| patient_initiated | Check for "must be initiated by patient" or "patient-initiated" |
| time_documentation | Default to inherited from global rules on time documentation |

**Confidence Levels:**

| Source | Confidence |
|--------|------------|
| Explicit in notes | high |
| Inherited from section | high |
| Inherited from global | medium |
| Inferred from description | medium |
| Inferred from context | low |

**Output:** `codes_enriched.json` (codes_raw + attributes)

### Pass 7: Consolidation & Validation

**Purpose:** Merge all outputs, validate consistency, flag issues.

**Process:**
```
1. Merge all JSON outputs into schema structure
2. Validate referential integrity:
   - All exclusion codes exist
   - All relationship targets exist
   - All rule references resolve
3. Flag anomalies:
   - Fees outside expected range
   - Codes appearing in multiple sections differently
   - Unresolved references
4. Generate validation report
5. Output final reference file
```

**Validation Checks:**

| Check | Action if Failed |
|-------|------------------|
| Exclusion references non-existent code | Flag for review |
| Fee > $1000 or < $1 | Flag for review |
| Code appears twice with different fees | Keep both, flag |
| Rule reference not found | Note as unresolved |
| Section boundary overlap | Merge or flag |

**Output:** `{province}_reference.json` (final) and `{province}_validation_report.json`

---

## 6. Phase 2: Code Matching

### Overview

Matching identifies which codes in the target province correspond to a given Alberta code. This uses the existing matching logic but now operates against the structured reference.

### Input

1. Alberta code definition (code, description, fee, clinical definition)
2. Province reference database (`{province}_reference.json`)

### Process

```
1. Load province reference
2. Filter to relevant sections (e.g., virtual care sections for telehealth codes)
3. Build context from:
   - Relevant section preambles
   - All codes in relevant sections with descriptions
   - Applicable modifiers
4. Send to LLM with matching prompt (similar to existing code)
5. Parse matches
6. Validate matches exist in reference
7. Enrich with reference data
```

### Matching Prompt

The existing prompt from `crosswalk_telehealth_all_provincesV3.py` works well. Key enhancement: include structured reference data instead of raw PDF text.

```
You are a senior physician billing specialist mapping Alberta fee codes to {province_name} equivalents.

ALBERTA CODE TO MATCH:
- Code: {ab_code}
- Description: {ab_description}
- Fee: ${ab_fee}

CLINICAL SERVICE DEFINITION:
{ab_clinical_definition}

{PROVINCE_NAME} REFERENCE DATA:

RELEVANT SECTIONS:
{formatted_section_list}

CODES IN VIRTUAL CARE / VISITS SECTIONS:
{formatted_code_list_with_descriptions_and_fees}

APPLICABLE MODIFIERS:
{formatted_modifier_list}

TASK:
Identify {province_name} codes that bill for the same clinical encounter.

[... rest of existing prompt ...]
```

### Output

`matched_codes_{ab_code}.json`

```json
{
  "ab_code": "03.03CV",
  "ab_description": "Telehealth consultation",
  "ab_fee": 25.09,
  "matches": [
    {
      "province": "MB",
      "code": "8340",
      "match_type": "primary",
      "match_confidence": "high",
      "match_reasoning": "Direct equivalent for episodic telephone virtual visit"
    },
    {
      "province": "MB",
      "code": "8321",
      "match_type": "primary",
      "match_confidence": "high",
      "match_reasoning": "Higher-value virtual visit covering both telephone and video"
    }
  ]
}
```

---

## 7. Phase 3: Ontology Population

### Overview

Ontology population enriches matched codes with comparable attributes by looking up data from the reference database. This phase involves minimal AI inference—it's primarily structured lookup.

### Input

1. Matched codes from Phase 2
2. Province reference database
3. Alberta code attributes

### Process

```
For each matched code:
    1. Look up code in reference.codes
    2. Look up code attributes in reference.code_attributes
    3. Look up applicable modifiers in reference.modifiers
    4. Look up exclusions in reference.exclusions
    5. Look up relationships in reference.relationships
    6. Calculate comparison metrics
    7. Compile final record
```

### Comparison Calculations

```python
# Fee comparison
fee_ratio = matched_fee / ab_fee
fee_difference = matched_fee - ab_fee

# Modality comparison
modality_match = (matched_modality == ab_modality) or 
                 (matched_modality == "both" and ab_modality in ["telephone", "video"])

# Time requirement comparison
if matched_min_time is None and ab_min_time is None:
    time_comparison = "same"
elif matched_min_time is None:
    time_comparison = "target_less_restrictive"
elif ab_min_time is None:
    time_comparison = "target_more_restrictive"
elif matched_min_time > ab_min_time:
    time_comparison = "target_more_restrictive"
elif matched_min_time < ab_min_time:
    time_comparison = "target_less_restrictive"
else:
    time_comparison = "same"

# Comparability score (simple weighted average)
score = 0
if fee_ratio > 0.8 and fee_ratio < 1.2:
    score += 30
if modality_match:
    score += 30
if time_comparison == "same":
    score += 20
if len(key_differences) < 3:
    score += 20
```

### Output

Final crosswalk output per Alberta code:

`crosswalk_{ab_code}.json`

```json
{
  "alberta_source": {
    "code": "03.03CV",
    "description": "Telehealth consultation",
    "fee": 25.09,
    "modality": "both",
    "minimum_time": 10,
    "patient_initiated": true,
    "exclusions": ["03.01AD", "03.01S", "03.01T", "..."]
  },
  "matched_codes": [
    {
      "province": "MB",
      "code": "8340",
      "description": "Episodic virtual visit by phone",
      "base_fee": 20.40,
      "match_type": "primary",
      "match_confidence": "high",
      "comparison": {
        "fee_ratio": 0.81,
        "fee_difference": -4.69,
        "modality_match": false,
        "modality_detail": "MB 8340 is telephone only; AB 03.03CV covers both",
        "time_comparison": "target_less_restrictive",
        "time_detail": "MB 8340 has no stated minimum; AB requires 10 min",
        "key_differences": [
          "Telephone only (no video)",
          "No minimum time stated",
          "Lower fee ($20.40 vs $25.09)"
        ],
        "comparability_score": 65,
        "comparability_notes": "Good match for telephone-only encounters; for video, use 8321"
      },
      "applicable_modifiers": [
        {
          "code": "5530",
          "name": "Extended clinic hours (evening)",
          "value": 0.20,
          "unit": "percentage",
          "conditions": "0600-0800 or 1700-2359 Mon-Thu"
        }
      ],
      "exclusions": [],
      "provenance": {
        "reference_version": "MB_2024-04-01_v1",
        "source_pages": [85],
        "rule_references": []
      }
    }
  ]
}
```

---

## 8. Implementation Details

### Technology Stack

| Component | Recommended Technology |
|-----------|------------------------|
| PDF Processing | pdfplumber (Python) |
| LLM API | OpenAI GPT-4 or Anthropic Claude |
| Data Storage | JSON files (SQLite for production) |
| Orchestration | Python scripts |
| Output | JSON → Excel via pandas/openpyxl |

### LLM Configuration

```python
# Recommended settings
MODEL = "gpt-4o"  # or "claude-sonnet-4-20250514"
TEMPERATURE = 0.1  # Low for extraction tasks
MAX_TOKENS = 4000  # Sufficient for most extractions

# For matching
MATCHING_TEMPERATURE = 0.2  # Slightly higher for reasoning
```

### Error Handling

| Error Type | Handling |
|------------|----------|
| LLM JSON parse failure | Retry with explicit JSON instructions |
| Missing page in PDF | Log and skip, note in validation |
| Code not found in reference | Flag match as "unverified" |
| Rate limit | Exponential backoff |
| Timeout | Retry with smaller batch |

### Cost Estimation

Assuming GPT-4o pricing ($2.50/1M input, $10/1M output):

| Phase | Tokens/Province | Est. Cost |
|-------|-----------------|-----------|
| Pass 1 (Structure) | ~100K input | $0.25 |
| Pass 2 (Rules) | ~500K input, 50K output | $1.75 |
| Pass 3 (Section) | ~200K input, 30K output | $0.80 |
| Pass 4 (Codes) | ~2M input, 200K output | $7.00 |
| Pass 5-7 | ~500K input, 100K output | $2.25 |
| **Total per province** | | **~$12** |
| **Total all 4 provinces** | | **~$48** |

Phase 2-3 (matching + ontology) per Alberta code: ~$0.50-1.00

---

## 9. File Structure

### Project Directory

```
provincial-billing-crosswalk/
├── README.md                          # This file
├── config/
│   ├── provinces.json                 # Province metadata
│   └── prompts/                       # LLM prompt templates
│       ├── structure_mapping.txt
│       ├── rules_extraction.txt
│       ├── section_rules.txt
│       ├── code_extraction.txt
│       └── matching.txt
├── data/
│   ├── source_pdfs/                   # Input PDFs
│   │   ├── MB_Payment_Schedule_April_2024.pdf
│   │   ├── ON_Schedule_Benefits_Feb_2024.pdf
│   │   └── ...
│   ├── references/                    # Phase 1 outputs
│   │   ├── MB/
│   │   │   ├── document_map.json
│   │   │   ├── global_rules.json
│   │   │   ├── section_rules.json
│   │   │   ├── codes_raw.json
│   │   │   ├── codes_enriched.json
│   │   │   ├── exclusions.json
│   │   │   ├── relationships.json
│   │   │   ├── MB_reference.json      # Final consolidated
│   │   │   └── MB_validation_report.json
│   │   ├── ON/
│   │   ├── SK/
│   │   └── BC/
│   ├── alberta/                       # Alberta code definitions
│   │   └── codes/
│   │       ├── 03.03CV.json
│   │       └── ...
│   ├── matches/                       # Phase 2 outputs
│   │   └── 03.03CV/
│   │       └── matched_codes.json
│   └── crosswalks/                    # Phase 3 outputs
│       └── 03.03CV/
│           ├── crosswalk.json
│           └── crosswalk.xlsx
├── scripts/
│   ├── phase1/
│   │   ├── 01_structure_mapping.py
│   │   ├── 02_rules_extraction.py
│   │   ├── 03_section_rules.py
│   │   ├── 04_code_extraction.py
│   │   ├── 05_relationship_extraction.py
│   │   ├── 06_attribute_inference.py
│   │   └── 07_consolidation.py
│   ├── phase2/
│   │   └── match_codes.py
│   ├── phase3/
│   │   └── populate_ontology.py
│   ├── utils/
│   │   ├── pdf_utils.py
│   │   ├── llm_utils.py
│   │   └── validation.py
│   └── run_all.py                     # Master orchestrator
└── output/
    └── final/
        └── crosswalk_03.03CV_all_provinces.xlsx
```

### Configuration Files

**provinces.json:**

```json
{
  "MB": {
    "name": "Manitoba",
    "document_pattern": "MB_Payment_Schedule",
    "code_pattern": "^\\d{4}$",
    "fee_pattern": "\\$?\\d+\\.\\d{2}"
  },
  "ON": {
    "name": "Ontario",
    "document_pattern": "Schedule.*Benefits",
    "code_pattern": "^[A-Z]\\d{3}[A-Z]?$",
    "fee_pattern": "\\d+\\.\\d{2}"
  }
}
```

---

## 10. Quality Assurance

### Validation Checkpoints

| Checkpoint | When | What to Check |
|------------|------|---------------|
| Structure mapping | After Pass 1 | Section boundaries make sense; page counts reasonable |
| Rules extraction | After Pass 2 | Key rules present (consultations, time, exclusions) |
| Code extraction | After Pass 4 | Code count reasonable; fees in expected range |
| Reference consolidation | After Pass 7 | No orphan references; validation report clean |
| Matching | After Phase 2 | Matches exist in reference; reasoning sensible |
| Final output | After Phase 3 | Fees populated; comparisons calculate correctly |

### Manual Review Triggers

Flag for manual review when:

- Fee ratio < 0.5 or > 2.0 (unusual fee difference)
- Comparability score < 50
- Code appears in validation report
- LLM confidence is "low" or "partial"
- Multiple equally-valid matches exist

### Sample Verification

For initial deployment, manually verify:

1. 5 random codes from each province reference
2. All matched codes for first 3 Alberta codes
3. All flagged items in validation reports

### Regression Testing

When updating extraction:

1. Re-run on same PDF
2. Compare code counts (should be identical)
3. Compare fee values (should be identical)
4. Review any differences

---

## Appendix A: Provincial Code Patterns

| Province | Code Format | Examples |
|----------|-------------|----------|
| Alberta | XX.XXYZ | 03.03CV, 03.01AD |
| Ontario | AXXX, AXXXA | A101, A102A, K130 |
| Manitoba | XXXX | 8340, 8321, 5530 |
| Saskatchewan | XXXY | 805B, 807E, 875A |
| British Columbia | PXXXXX | P13037, P13017 |

## Appendix B: Common Rule Types by Province

| Rule Type | MB | ON | SK | BC |
|-----------|-----|-----|-----|-----|
| Consultation definition | Rule 7-10 | Preamble | Part 1 | Preamble |
| Time documentation | Rule 62 | General Preamble | Part 2 | Section rules |
| Hospital premium | Section preambles | Facility fees | Section rules | Varies |
| After-hours premium | General Schedule | K codes | Section-specific | Varies |
| Virtual care rules | Section notes | Appendix J | Part 5 | Telehealth section |

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| Add-on code | Code billed in addition to a primary code |
| Base fee | The standard fee before any modifiers |
| Crosswalk | Mapping between equivalent codes in different systems |
| Exclusion | Codes that cannot be billed together |
| Modifier | Code or rule that adjusts the base fee |
| Ontology | Structured classification of code attributes |
| Preamble | Introductory rules section of fee schedule |
| Primary code | Main code for a service (vs. add-on) |
| Reference database | Structured extract of a fee schedule |
| Tariff | Manitoba term for billing code |

---

**Document version:** 1.0  
**Last updated:** January 2026  
**Authors:** HelpSeeker Technologies
