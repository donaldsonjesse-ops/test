# Provincial Billing Code Crosswalk System

## Technical Documentation

### Version: 3.03 All Province Alberta Crosswalk

---

## 1. System Overview

This notebook implements a two-phase LLM-based extraction system for mapping Alberta Health Services billing codes to equivalent codes across four Canadian provincial fee schedules: British Columbia (BC), Manitoba (MB), Ontario (ON), and Saskatchewan (SK).

The system processes provincial physician payment schedule PDFs, extracts relevant billing codes that match a specified Alberta source code, and enriches the extracted data with detailed billing attributes.

### 1.1 Current Implementation

The current implementation uses Alberta billing code `03.03CV` (Telehealth consultation, $25.09) as the source code for crosswalk mapping. This configuration is hardcoded in Cell 4 (`ALBERTA_CODE_CONFIG`). In production, this should be connected to an external file containing all Alberta billing codes and their associated metadata (code, description, fee, clinical_definition, service_context, search_criteria, exclusion_criteria).

---

## 2. Architecture

### 2.1 Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INPUT FILES                                   │
├─────────────────────────────────────────────────────────────────────┤
│  - Provincial PDFs (4)          BC, MB, ON, SK Payment Schedules    │
│  - Section Reference CSVs (4)   Page-to-section mapping per province│
│  - Extraction Taxonomy          Attribute definitions (Excel)        │
│  - Alberta Code Config          Source code to match (Cell 4)        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     PHASE 1: CODE EXTRACTION                         │
├─────────────────────────────────────────────────────────────────────┤
│  For each province:                                                  │
│  1. Load PDF pages into memory (pdfplumber)                         │
│  2. Build section chunks using reference CSV                         │
│  3. For each section chunk:                                          │
│     - Build province-specific prompt with Alberta code context       │
│     - Send to GPT-4.1 API                                            │
│     - Parse JSON response for matching codes                         │
│  4. Collect primary_codes and add_on_codes                          │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   PHASE 2: ATTRIBUTE EXTRACTION                      │
├─────────────────────────────────────────────────────────────────────┤
│  For each extracted code:                                            │
│  1. Load full rules/preamble text from PDF (PyMuPDF)                │
│  2. Build attribute extraction prompt with:                          │
│     - Code info from Phase 1                                         │
│     - Section chunk text (truncated to 30K chars)                   │
│     - Full rules text (no truncation)                               │
│     - Taxonomy reference                                             │
│  3. Extract: modality, time_requirements, frequency_limits,         │
│              same_day_exclusions, premiums, additional_notes         │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         OUTPUT FILES                                 │
├─────────────────────────────────────────────────────────────────────┤
│  - 3.02_BC_Alberta_Complete.xlsx                                    │
│  - 3.02_MB_Alberta_Complete.xlsx                                    │
│  - 3.02_ON_Alberta_Complete.xlsx                                    │
│  - 3.02_SK_Alberta_Complete.xlsx                                    │
│  - 3.03_All_Province_{AB_CODE}_Complete.xlsx (combined)             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Input File Specifications

### 3.1 Provincial PDF Files

| Province | Expected Filename Pattern | Document |
|----------|--------------------------|----------|
| BC | `*BC*.pdf` | BC Payment Schedule |
| MB | `*MB*.pdf` | MB Payment Schedule |
| ON | `*ON*.pdf` | ON Schedule of Benefits |
| SK | `*SK*.pdf` | SK Payment Schedule |

Auto-detection logic in Cell 2a matches filenames containing province codes (case-insensitive).

### 3.2 Section Reference CSVs

Each province requires a section reference CSV that maps the PDF structure. Required columns:

| Column | Type | Description |
|--------|------|-------------|
| `level_1` | string | Primary section name (e.g., "GENERAL PRACTICE", "INTERNAL MEDICINE") |
| `level_2` | string | Subsection name (Ontario only, nullable) |
| `page_start` | integer | Starting page number for this section |

Expected filenames:
- `bc_section_reference_simple.csv`
- `manitoba_section_reference_final.csv`
- `on_section_reference_full.csv`
- `sk_section_reference_simple.csv`

**Note:** Each provincial PDF should be indexed into a master section reference sheet to improve extraction efficiency and efficacy. This indexing process maps the hierarchical structure of each fee schedule (sections, subsections, page ranges) enabling precise chunking for LLM processing.

### 3.3 Extraction Taxonomy

Excel file (`extraction_taxonomy.xlsx`) defining Phase 2 attributes. Required columns:

| Column | Description |
|--------|-------------|
| `attribute` | Attribute name (e.g., "modality", "minimum_time_minutes") |
| `data_type` | Expected type (string, integer, array, boolean) |
| `definition` | Human-readable definition for LLM context |
| `taxonomy` | Controlled vocabulary or valid values |

**Note:** The current extraction taxonomy attributes require a complete overhaul. The existing attribute definitions do not adequately capture the complexity of provincial billing rules, resulting in inconsistent extraction quality across provinces.

---

## 4. Configuration Objects

### 4.1 ALBERTA_CODE_CONFIG (Cell 4)

Defines the source Alberta billing code for crosswalk mapping.

```python
ALBERTA_CODE_CONFIG = {
    'code': str,                    # Alberta HSC code (e.g., "03.03CV")
    'description': str,             # Short description
    'fee': float,                   # Base fee amount
    'clinical_definition': str,     # Detailed clinical service definition
    'service_context': str,         # LLM context for matching logic
    'search_criteria': str,         # What to look for in provincial schedules
    'exclusion_criteria': str,      # What to exclude from matches
}
```

**Example (03.03CV - Telehealth consultation):**
```python
'code': '03.03CV'
'description': 'Telehealth consultation'
'fee': 25.09
'clinical_definition': """Assessment of a patient's condition via telephone
    or secure videoconference. Total physician time must be MINIMUM 10 MINUTES..."""
'search_criteria': """Virtual visits, Telephone consultations, Video consultations,
    Telehealth codes, Patient-facing virtual encounters"""
'exclusion_criteria': """Physician-to-physician consultations, E-consults,
    In-person only codes, Diagnostic procedures"""
```

### 4.2 PROVINCE_CONFIGS (Cell 5)

Static configuration for each province's fee schedule structure.

```python
PROVINCE_CONFIGS[prov_code] = {
    'name': str,                    # Full province name
    'chunking_level': int,          # 1 = Level 1 sections, 2 = Level 2 (ON only)
    'rules_pages': tuple(int, int), # Start/end pages for preamble/rules
    'skip_sections': list[str],     # Section names to exclude from processing
    'min_clinical_page': int,       # (MB, SK) First page of clinical content
    'special_fields': list[str],    # Province-specific output columns
    'extraction_rules': str,        # Province-specific LLM instructions
    'json_schema': dict,            # Expected JSON response structure
    'output_columns': list[str],    # Final DataFrame column order
}
```

#### Province-Specific Parameters

| Province | Chunking | Rules Pages | Special Fields | Notes |
|----------|----------|-------------|----------------|-------|
| BC | Level 1 | 1-52 | None | Code prefixes: P, G, PG |
| MB | Level 1 | 1-82 | None | Clinical content starts page 83 |
| ON | Level 2 | 1-126 | Fee_Type, Setting, Level_3_Heading | H/P columns, Surg/Asst/Anae fees |
| SK | Level 1 | 1-70 | Fee_Type, Age_Premium_Applies | Referred/Not Referred dual-fees |

---

## 5. Core Functions (Cell 6)

### 5.1 PDF Processing

| Function | Parameters | Returns | Description |
|----------|------------|---------|-------------|
| `load_pdf_pages(pdf_path)` | str | dict[int, str], int | Loads all pages into memory using pdfplumber. Returns {page_num: text} and total_pages. |
| `extract_rules_text(pdf_path, rules_pages)` | str, tuple | str | Extracts preamble/rules text using PyMuPDF for full-text extraction. |

### 5.2 Section Chunking

| Function | Parameters | Returns | Description |
|----------|------------|---------|-------------|
| `build_section_chunks_level1(df_ref, pdf_pages, total_pages, prov_config)` | DataFrame, dict, int, dict | dict | Builds section chunks for BC, MB, SK using Level 1 sections. |
| `build_section_chunks_level2(df_ref, pdf_pages, total_pages, prov_config)` | DataFrame, dict, int, dict | dict | Builds section chunks for ON using Level 2 subsections. |

**Section Chunk Structure:**
```python
section_chunks[section_key] = {
    'text': str,          # Concatenated page text with "=== PAGE X ===" markers
    'level_1': str,       # Level 1 section name
    'level_2': str,       # Level 2 subsection name (or same as level_1)
    'start_page': int,    # First page of section
    'end_page': int,      # Last page of section
    'page_count': int,    # Number of pages
    'char_count': int,    # Total character count
}
```

### 5.3 Token Management

| Function | Parameters | Returns | Description |
|----------|------------|---------|-------------|
| `get_dynamic_max_tokens(char_count)` | int | int | Phase 1: Returns max_completion_tokens based on section size (4000-20000). |
| `get_phase2_max_tokens(rules_char_count)` | int | int | Phase 2: Returns max_completion_tokens based on rules size (2500-4000). |

**Phase 1 Token Allocation:**
| Section Char Count | max_completion_tokens |
|-------------------|----------------------|
| >150,000 | 20,000 |
| >80,000 | 14,000 |
| >40,000 | 10,000 |
| >15,000 | 6,000 |
| ≤15,000 | 4,000 |

### 5.4 Prompt Builders

| Function | Parameters | Returns | Description |
|----------|------------|---------|-------------|
| `build_phase1_prompt(section_key, section_info, prov_code, prov_config)` | str, dict, str, dict | str | Builds code extraction prompt with Alberta code context and province-specific rules. |
| `build_phase2_prompt(code_info, chunk_text, rules_text, prov_code)` | dict, str, str, str | str | Builds attribute extraction prompt. Rules text is NOT truncated; chunk_text truncated to 30K chars. |

### 5.5 Result Processing

| Function | Parameters | Returns | Description |
|----------|------------|---------|-------------|
| `process_phase1_result(result, section_key, section_info, prov_code, code_chunks)` | dict, str, dict, str, dict | list[dict] | Parses Phase 1 JSON into standardized rows. Handles province-specific schemas (ON uses 'codes', others use 'primary_codes'/'add_on_codes'). |

---

## 6. Output Schema

### 6.1 Common Columns (All Provinces)

| Column | Type | Source | Description |
|--------|------|--------|-------------|
| `AB_Code` | string | Config | Alberta source code |
| `AB_Description` | string | Config | Alberta code description |
| `AB_Fee` | float | Config | Alberta base fee |
| `Target_Province` | string | Processing | Province code (BC/MB/ON/SK) |
| `Code` | string | Phase 1 | Extracted provincial code |
| `Description` | string | Phase 1 | Full code description |
| `Fee` | string | Phase 1 | Fee amount (may include modifiers) |
| `Type` | string | Phase 1 | PRIMARY or ADD-ON |
| `Modality` | string | Phase 1/2 | telephone, video, both, or empty |
| `Links_To` | string | Phase 1 | Related codes (for add-ons) |
| `Condition` | string | Phase 1 | Billing conditions |
| `Reasoning` | string | Phase 1 | LLM reasoning for match |
| `Level_1_Section` | string | Phase 1 | Top-level section name |
| `Level_2_Subsection` | string | Phase 1 | Subsection name |
| `Page_Found` | integer | Phase 1 | PDF page number |

### 6.2 Province-Specific Columns

**Ontario (ON):**
| Column | Type | Description |
|--------|------|-------------|
| `Fee_Type` | string | Standard, Surgeon, Assistant, Anaesthesia |
| `Setting` | string | Hospital, Professional, N/A |
| `Level_3_Heading` | string | Third-level subsection |

**Saskatchewan (SK):**
| Column | Type | Description |
|--------|------|-------------|
| `Fee_Type` | string | Referred, Not Referred, Standard |
| `Age_Premium_Applies` | boolean | Whether age premium (0-5, 65+) applies |

### 6.3 Phase 2 Attribute Columns

| Column | Type | Description |
|--------|------|-------------|
| `modality` | string | telephone, video, both, in_person, asynchronous |
| `minimum_time_minutes` | integer | Minimum physician time required |
| `frequency_per_day` | integer | Maximum claims per day |
| `frequency_per_year` | integer | Maximum claims per year |
| `frequency_per_year_period` | string | annual, quarterly, 90_days, monthly |
| `same_day_exclusions` | string | Comma-separated codes that cannot be billed same day |
| `premium_extended_hours` | string | Extended hours premium (rate, code, conditions) |
| `premium_location` | string | Location-based premium |
| `premium_age` | string | Age-based premium |
| `premium_other` | string | Other applicable premiums |
| `additional_notes` | string | Other billing information |

---

## 7. API Configuration

### 7.1 Model Settings

| Parameter | Value | Notes |
|-----------|-------|-------|
| Model | `gpt-4.1-2025-04-14` | OpenAI GPT-4.1 |
| Temperature | 0.1 | Low temperature for consistent extraction |
| max_completion_tokens | Dynamic | See Token Management section |

### 7.2 Cost Tracking

```python
# GPT-4 pricing estimate
input_cost = (prompt_tokens / 1_000_000) * $3.00
output_cost = (completion_tokens / 1_000_000) * $15.00
total_cost = input_cost + output_cost
```

---

## 8. Execution Flow

### 8.1 Per-Province Processing (Cells 7a-7d)

```
1. Validate input files exist (PDF + Reference CSV)
2. Reset cost tracking
3. Load reference CSV, sort by page_start
4. Load PDF pages into memory
5. Build section chunks (Level 1 or Level 2)
6. PHASE 1: For each section chunk:
   a. Calculate dynamic token limit
   b. Build extraction prompt
   c. Call GPT-4.1 API
   d. Parse JSON response
   e. Process results into standardized rows
   f. Store chunk text for Phase 2
7. PHASE 2: For each extracted code:
   a. Load full rules text
   b. Calculate Phase 2 token limit
   c. Build attribute extraction prompt
   d. Call GPT-4.1 API
   e. Parse and merge attribute results
8. Combine Phase 1 + Phase 2 DataFrames
9. Export to Excel
10. Download file
```

### 8.2 Combining Results (Cell 8)

```
1. Collect all province DataFrames (df_bc, df_mb, df_on, df_sk)
2. Concatenate with ignore_index=True
3. Fill missing columns across provinces
4. Export combined file: 3.03_All_Province_{AB_CODE}_Complete.xlsx
5. Print summary statistics
6. Download combined file
```

---

## 9. JSON Response Schemas

### 9.1 Phase 1 - BC/MB (Level 1)

```json
{
  "section_name": "string",
  "found": true,
  "primary_codes": [
    {
      "code": "string",
      "description": "string",
      "fee": "string",
      "modality": "telephone|video|both",
      "page_found": 123,
      "section_heading": "string",
      "reasoning": "string"
    }
  ],
  "add_on_codes": [
    {
      "code": "string",
      "description": "string",
      "fee": "string",
      "modality": "telephone|video|both",
      "page_found": 123,
      "section_heading": "string",
      "links_to": ["code1", "code2"],
      "condition": "string"
    }
  ]
}
```

### 9.2 Phase 1 - Ontario (Level 2)

```json
{
  "section_key": "string",
  "found": true,
  "codes": [
    {
      "code": "string",
      "description": "string",
      "fee": "string",
      "fee_type": "Standard|Surgeon|Assistant|Anaesthesia",
      "setting": "Hospital|Professional|N/A",
      "modality": "telephone|video|both",
      "page_found": 123,
      "level_3_heading": "string",
      "is_addon": false,
      "links_to": [],
      "condition": "",
      "reasoning": "string"
    }
  ]
}
```

### 9.3 Phase 1 - Saskatchewan

```json
{
  "section_name": "string",
  "found": true,
  "primary_codes": [
    {
      "code": "string",
      "description": "string",
      "fee": "string",
      "fee_type": "Referred|Not Referred|Standard",
      "modality": "telephone|video|both",
      "page_found": 123,
      "section_heading": "string",
      "age_premium_applies": true,
      "reasoning": "string"
    }
  ],
  "add_on_codes": [...]
}
```

### 9.4 Phase 2 - Attribute Extraction

```json
{
  "modality": "telephone|video|both|in_person|asynchronous|null",
  "minimum_time_minutes": 10,
  "frequency_per_day": 1,
  "frequency_per_year": null,
  "frequency_per_year_period": "annual|quarterly|90_days|monthly|null",
  "same_day_exclusions": ["code1", "code2"],
  "premium_extended_hours": "50% K001 after 5pm weekdays",
  "premium_location": null,
  "premium_age": "30% for patients ≤12 years",
  "premium_other": null,
  "additional_notes": "string"
}
```

---

## 10. Dependencies

```
openai>=1.0.0      # OpenAI API client
pandas>=2.0.0      # DataFrame operations
pdfplumber>=0.9.0  # PDF text extraction (page loading)
PyMuPDF>=1.22.0    # PDF text extraction (rules extraction)
openpyxl>=3.1.0    # Excel export
tqdm>=4.65.0       # Progress bars
```

---

## 11. File Structure

```
project/
├── 3.03 All Province_Alberta Crosswalk.ipynb    # Main notebook
├── 3.03 All Province_Alberta Crosswalk_Truncated.ipynb  # Fast test version
├── extraction_taxonomy.xlsx                      # Attribute definitions
├── bc_section_reference_simple.csv              # BC section index
├── manitoba_section_reference_final.csv         # MB section index
├── on_section_reference_full.csv                # ON section index
├── sk_section_reference_simple.csv              # SK section index
└── resources/
    └── schedules/
        ├── BC_Physician_Manual.pdf
        ├── MB_Physician_Manual.pdf
        ├── ON_Schedule_of_Benefits_2024.pdf
        └── SK_Fee_Schedule_2025.pdf
```

---

## 12. Known Limitations

### 12.1 Alberta Code Configuration
The current implementation has the Alberta code hardcoded in Cell 4. For production use, this should be connected to an external file (CSV/Excel) containing all Alberta billing codes with their complete metadata.

### 12.2 Section Reference Dependencies
Each provincial PDF must have a corresponding section reference CSV. These CSVs must be manually created by indexing each PDF's table of contents and section structure. Each provincial PDF should be indexed into a master section reference sheet to improve extraction efficiency and efficacy.

### 12.3 Extraction Taxonomy
The attributes defined in `extraction_taxonomy.xlsx` require a complete overhaul. Current limitations include:
- Inconsistent attribute granularity across provinces
- Missing province-specific attributes
- Ambiguous definitions leading to extraction inconsistencies
- Lack of validation rules for extracted values

### 12.4 Phase 1 TASK Line
The Phase 1 prompt contains a hardcoded TASK description ("patient-facing virtual assessments"). For non-telehealth codes, the `task_description` field in `ALBERTA_CODE_CONFIG` must be added and the prompt updated to use `{ab['task_description']}`.

---

## 13. Version History

| Version | Date | Changes |
|---------|------|---------|
| 3.03 | 2024 | Consolidated all-province notebook |
| 3.02 | 2024 | Individual province notebooks |
| 3.01 | 2024 | Initial two-phase extraction |
