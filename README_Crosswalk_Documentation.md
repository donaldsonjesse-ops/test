# Provincial Billing Code Crosswalk System

## Technical Documentation

### Version: 3.03 All Province Alberta Crosswalk
### Platform: Google Colab (Python 3.10+)

---

## 1. System Overview

### 1.1 Purpose

This system implements an automated two-phase LLM-based extraction pipeline for mapping Alberta Health Services (AHS) physician billing codes to equivalent billing codes across four Canadian provincial fee schedules:

- **BC** - British Columbia Medical Services Plan (MSP) Payment Schedule
- **MB** - Manitoba Physician's Manual
- **ON** - Ontario Schedule of Benefits for Physician Services
- **SK** - Saskatchewan Payment Schedule for Insured Services Provided by a Physician

The system addresses the challenge of interprovincial billing code harmonization, enabling healthcare organizations to identify equivalent services across jurisdictions for revenue cycle management, cross-provincial billing analysis, and fee schedule comparison.

### 1.2 Current Implementation State

The current implementation uses Alberta billing code `03.03CV` (Telehealth consultation, $25.09, Category V Visit) as the hardcoded source code for crosswalk mapping. This configuration resides in Cell 4 (`ALBERTA_CODE_CONFIG`).

**Production Requirement:** This should be refactored to connect to an external master file (CSV or Excel) containing all Alberta billing codes with their complete metadata structure:
- `code` - Health Service Code (HSC)
- `description` - Service description
- `fee` - Base fee amount
- `clinical_definition` - Full clinical service definition
- `service_context` - Contextual information for LLM matching
- `search_criteria` - Positive matching criteria
- `exclusion_criteria` - Negative matching criteria
- `task_description` - Dynamic TASK line content for Phase 1 prompts

### 1.3 Processing Model

The system employs a **two-phase extraction model**:

**Phase 1 (Code Discovery):** Identifies all provincial billing codes within each section of the fee schedule that match the clinical definition and service characteristics of the Alberta source code. This phase processes the PDF in section-level chunks and returns structured JSON with code identifiers, descriptions, fees, and matching rationale.

**Phase 2 (Attribute Enrichment):** For each code discovered in Phase 1, extracts detailed billing attributes by analyzing both the code-specific section text and the full provincial rules/preamble. This phase captures time requirements, frequency limits, same-day exclusions, premium modifiers, and other billing constraints.

---

## 2. Architecture

### 2.1 High-Level Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INPUT LAYER                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │  Provincial PDFs    │  │  Section Reference  │  │    Extraction       │  │
│  │  (4 files)          │  │  CSVs (4 files)     │  │    Taxonomy         │  │
│  │                     │  │                     │  │                     │  │
│  │  - BC: ~400 pages   │  │  - level_1          │  │  - attribute        │  │
│  │  - MB: ~600 pages   │  │  - level_2          │  │  - data_type        │  │
│  │  - ON: ~900 pages   │  │  - page_start       │  │  - definition       │  │
│  │  - SK: ~350 pages   │  │                     │  │  - taxonomy         │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐                           │
│  │  Alberta Code       │  │  OpenAI API Key     │                           │
│  │  Configuration      │  │                     │                           │
│  │  (Cell 4)           │  │  GPT-4.1 Access     │                           │
│  └─────────────────────┘  └─────────────────────┘                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PREPROCESSING LAYER                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  PDF Page Loading (pdfplumber)                                      │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  Input:  PDF file path                                              │    │
│  │  Output: Dictionary {page_num: extracted_text}                      │    │
│  │  Method: pdfplumber.open() → page.extract_text()                    │    │
│  │  Note:   Progress bar via tqdm, stores all pages in memory          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Section Chunking                                                    │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  Level 1 (BC, MB, SK):                                              │    │
│  │    - Groups pages by level_1 section from reference CSV             │    │
│  │    - Each chunk = all pages from section start to next section      │    │
│  │    - Respects min_clinical_page for MB/SK                           │    │
│  │                                                                      │    │
│  │  Level 2 (ON only):                                                 │    │
│  │    - Uses both level_1 and level_2 from reference CSV               │    │
│  │    - More granular chunks for Ontario's complex structure           │    │
│  │    - Section key = "level_1 | level_2"                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Rules/Preamble Extraction (PyMuPDF/fitz)                           │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  Input:  PDF path, rules_pages tuple (start, end)                   │    │
│  │  Output: Concatenated rules text with page markers                  │    │
│  │  Method: fitz.open() → page.get_text()                              │    │
│  │  Note:   Separate from pdfplumber for different extraction needs    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PHASE 1: CODE EXTRACTION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  For each section_chunk in section_chunks:                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  1. Calculate Dynamic Token Limit                                   │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     char_count = len(section_chunk['text'])                         │    │
│  │     if char_count > 150000: max_tokens = 20000                      │    │
│  │     elif char_count > 80000: max_tokens = 14000                     │    │
│  │     elif char_count > 40000: max_tokens = 10000                     │    │
│  │     elif char_count > 15000: max_tokens = 6000                      │    │
│  │     else: max_tokens = 4000                                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  2. Build Phase 1 Prompt                                            │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     Components:                                                     │    │
│  │     - Alberta code context (code, description, fee, definition)     │    │
│  │     - Service context and matching guidance                         │    │
│  │     - Section location (level_1 > level_2, pages X to Y)            │    │
│  │     - Full section text with page markers                           │    │
│  │     - TASK instruction (currently hardcoded for telehealth)         │    │
│  │     - Province-specific extraction rules                            │    │
│  │     - Search criteria (what to look for)                            │    │
│  │     - Exclusion criteria (what to skip)                             │    │
│  │     - Province-specific JSON schema template                        │    │
│  │     - No-match fallback JSON                                        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  3. API Call                                                        │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     client.chat.completions.create(                                 │    │
│  │         model="gpt-4.1-2025-04-14",                                 │    │
│  │         messages=[{"role": "user", "content": prompt}],             │    │
│  │         temperature=0.1,                                            │    │
│  │         max_completion_tokens=max_tokens                            │    │
│  │     )                                                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  4. Response Parsing                                                │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     content = response.choices[0].message.content                   │    │
│  │     match = re.search(r'\{[\s\S]*\}', content)  # Extract JSON      │    │
│  │     result = json.loads(match.group())                              │    │
│  │     if result.get('found'):                                         │    │
│  │         rows = process_phase1_result(...)                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  5. Result Processing                                               │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     For each code in result:                                        │    │
│  │     - Generate unique_key for Phase 2 lookup                        │    │
│  │     - Store section text in code_chunks dict                        │    │
│  │     - Create standardized row dict with all columns                 │    │
│  │     - Append to all_results list                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Output: all_results (list of dicts), code_chunks (dict of section texts)   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PHASE 2: ATTRIBUTE EXTRACTION                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  1. Load Full Rules Text                                            │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     rules_text = extract_rules_text(pdf_path, rules_pages)          │    │
│  │     Note: NO TRUNCATION - full rules required for accurate          │    │
│  │           extraction of billing constraints                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  For each code_info in all_results:                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  2. Build Phase 2 Prompt                                            │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     Components:                                                     │    │
│  │     - Code info from Phase 1 (code, description, fee, type, section)│    │
│  │     - Taxonomy reference (attribute definitions)                    │    │
│  │     - Province-specific rules checklist (SK only)                   │    │
│  │     - Full rules/preamble text (NOT truncated)                      │    │
│  │     - Code-specific section text (truncated to 30,000 chars)        │    │
│  │     - Attribute extraction JSON schema                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  3. API Call                                                        │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     max_tokens = get_phase2_max_tokens(len(rules_text))             │    │
│  │     - rules > 300K chars: 4000 tokens                               │    │
│  │     - rules > 200K chars: 3000 tokens                               │    │
│  │     - rules ≤ 200K chars: 2500 tokens                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  4. Response Processing                                             │    │
│  │     ───────────────────────────────────────────────────────────    │    │
│  │     - Parse JSON from response                                      │    │
│  │     - Convert same_day_exclusions array to comma-separated string   │    │
│  │     - Store with unique_key for DataFrame merge                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Output: phase2_results (list of dicts with attributes)                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            OUTPUT LAYER                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  DataFrame Merge                                                    │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  df_phase1 = pd.DataFrame(all_results)                              │    │
│  │  df_phase2 = pd.DataFrame(phase2_results)                           │    │
│  │  df_final = df_phase1.merge(df_phase2, on='_unique_key', how='left')│    │
│  │  df_final = df_final.drop(columns=['_unique_key'])                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Per-Province Output                                                │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  3.02_BC_Alberta_Complete.xlsx                                      │    │
│  │  3.02_MB_Alberta_Complete.xlsx                                      │    │
│  │  3.02_ON_Alberta_Complete.xlsx                                      │    │
│  │  3.02_SK_Alberta_Complete.xlsx                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Combined Output                                                    │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  df_combined = pd.concat([df_bc, df_mb, df_on, df_sk])              │    │
│  │  3.03_All_Province_{AB_CODE}_Complete.xlsx                          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Input File Specifications

### 3.1 Provincial PDF Files

#### 3.1.1 File Detection Logic

The system auto-detects province from filename using case-insensitive substring matching in Cell 2a:

```python
for filename in uploaded_pdfs.keys():
    filename_upper = filename.upper()
    if 'BC' in filename_upper:
        PDF_FILES['BC'] = filename
    elif 'MB' in filename_upper:
        PDF_FILES['MB'] = filename
    elif 'ON' in filename_upper:
        PDF_FILES['ON'] = filename
    elif 'SK' in filename_upper:
        PDF_FILES['SK'] = filename
```

#### 3.1.2 Expected File Characteristics

| Province | Expected Pattern | Typical Size | Page Count | Document Title |
|----------|-----------------|--------------|------------|----------------|
| BC | `*BC*.pdf` | 15-25 MB | 350-450 | Payment Schedule |
| MB | `*MB*.pdf` | 10-20 MB | 500-700 | Physician's Manual |
| ON | `*ON*.pdf` | 25-40 MB | 800-1000 | Schedule of Benefits |
| SK | `*SK*.pdf` | 12-18 MB | 300-400 | Payment Schedule |

#### 3.1.3 PDF Structure Requirements

Each PDF must contain:
1. **Preamble/Rules Section** - General billing rules, definitions, modifiers (front matter)
2. **Clinical Sections** - Organized by specialty or service type
3. **Consistent Page Numbering** - For accurate section reference mapping

### 3.2 Section Reference CSVs

#### 3.2.1 Purpose

Section reference CSVs provide the structural index of each provincial PDF, mapping section names to starting page numbers. This enables the system to chunk the PDF into manageable sections for LLM processing.

**Critical Note:** Each provincial PDF should be indexed into a master section reference sheet to improve extraction efficiency and efficacy. This manual indexing process requires reviewing each PDF's table of contents and verifying page numbers match the actual content locations.

#### 3.2.2 Required Schema

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `level_1` | string | Yes | Primary section name (exact match to PDF heading) |
| `level_2` | string | No* | Subsection name (*Required for Ontario) |
| `page_start` | integer | Yes | First page number where section begins |

#### 3.2.3 File Detection Logic (Cell 2b)

```python
for filename in uploaded_refs.keys():
    filename_lower = filename.lower()
    if filename_lower.startswith('bc') or '_bc_' in filename_lower:
        REF_FILES['BC'] = filename
    elif 'mb' in filename_lower or 'manitoba' in filename_lower:
        REF_FILES['MB'] = filename
    elif filename_lower.startswith('sk') or '_sk_' in filename_lower:
        REF_FILES['SK'] = filename
    elif filename_lower.startswith('on') or '_on_' in filename_lower:
        REF_FILES['ON'] = filename
```

Note: Pattern matching order matters - SK is checked before ON to prevent false matches (e.g., "section" contains "on").

#### 3.2.4 Province-Specific Reference File Examples

**BC (bc_section_reference_simple.csv):**
```csv
level_1,page_start
3. GENERAL PRACTICE,53
4. INTERNAL MEDICINE,89
5. PAEDIATRICS,127
6. PSYCHIATRY,145
7. ANAESTHESIA,162
...
```

**MB (manitoba_section_reference_final.csv):**
```csv
level_1,page_start
GENERAL PRACTICE,83
INTERNAL MEDICINE,142
PAEDIATRICS,198
PSYCHIATRY,223
...
```

**ON (on_section_reference_full.csv):**
```csv
level_1,level_2,page_start
Consultations and Visits,General Listings,127
Consultations and Visits,Special Visit Premium,135
Consultations and Visits,Delegated Procedures,142
Diagnostic and Therapeutic Procedures,Anaesthesia,156
...
```

**SK (sk_section_reference_simple.csv):**
```csv
level_1,page_start
GENERAL PRACTICE,71
INTERNAL MEDICINE,112
PAEDIATRICS,145
PSYCHIATRY,167
...
```

### 3.3 Extraction Taxonomy

#### 3.3.1 Purpose

The extraction taxonomy Excel file (`extraction_taxonomy.xlsx`) defines the Phase 2 attributes that should be extracted for each billing code. It provides the LLM with attribute names, expected data types, definitions, and valid value taxonomies.

#### 3.3.2 Required Schema

| Column | Type | Description |
|--------|------|-------------|
| `attribute` | string | Attribute name (used as output column name) |
| `data_type` | string | Expected type: string, integer, float, boolean, array |
| `definition` | string | Human-readable definition for LLM context |
| `taxonomy` | string | Controlled vocabulary or valid values |

#### 3.3.3 Current Attributes

| Attribute | Data Type | Definition | Taxonomy |
|-----------|-----------|------------|----------|
| `modality` | string | Mode of service delivery | telephone, video, both, in_person, asynchronous |
| `minimum_time_minutes` | integer | Minimum physician time required | Any positive integer |
| `frequency_per_day` | integer | Maximum claims allowed per day | Any positive integer |
| `frequency_per_year` | integer | Maximum claims allowed per year | Any positive integer |
| `frequency_per_year_period` | string | Period for year-based frequency | annual, quarterly, 90_days, monthly |
| `same_day_exclusions` | array | Codes that cannot be billed same day | Array of code strings |
| `premium_extended_hours` | string | Extended hours premium details | Rate, code, conditions |
| `premium_location` | string | Location-based premium details | Rate, code, conditions |
| `premium_age` | string | Age-based premium details | Rate, conditions |
| `premium_other` | string | Other premium details | Rate, code, conditions |
| `additional_notes` | string | Other important billing info | Free text |

#### 3.3.4 Taxonomy String Generation (Cell 2c)

```python
taxonomy_reference = "\n".join([
    f"- {row['attribute']} ({row['data_type']}): {row['definition']} Taxonomy: {row['taxonomy']}"
    for _, row in df_taxonomy.iterrows()
])
```

This generates a formatted string included in every Phase 2 prompt.

**Critical Note:** The current extraction taxonomy attributes require a complete overhaul. The existing attribute definitions:
- Do not adequately capture the complexity of provincial billing rules
- Result in inconsistent extraction quality across provinces
- Lack province-specific attributes (e.g., Ontario's H/P settings, SK's referral status)
- Have ambiguous definitions leading to extraction inconsistencies
- Missing validation rules for extracted values

---

## 4. Configuration Objects

### 4.1 ALBERTA_CODE_CONFIG (Cell 4)

#### 4.1.1 Structure

```python
ALBERTA_CODE_CONFIG = {
    'code': str,                    # Alberta Health Service Code (HSC)
    'description': str,             # Short service description
    'fee': float,                   # Base fee in CAD
    'clinical_definition': str,     # Detailed clinical service definition (multi-line)
    'service_context': str,         # LLM guidance for matching logic
    'search_criteria': str,         # What to look for (positive criteria)
    'exclusion_criteria': str,      # What to exclude (negative criteria)
}
```

#### 4.1.2 Current Configuration (03.03CV Example)

```python
ALBERTA_CODE_CONFIG = {
    'code': '03.03CV',
    'description': 'Telehealth consultation',
    'fee': 25.09,

    'clinical_definition': """Assessment of a patient's condition via telephone or secure videoconference.

NOTE:
- At minimum: limited assessment requiring history related to presenting problems,
  appropriate records review, and advice to the patient
- Total physician time spent providing patient care must be MINIMUM 10 MINUTES
- If less than 10 minutes same day, must use HSC 03.01AD instead
- May only be claimed if service was initiated by the patient or their agent
- May only be claimed if service is personally rendered by the physician
- Benefit includes ordering appropriate diagnostic tests and discussion with patient
- Patient record must include detailed summary of all services including start/stop times
- Time spent on administrative tasks cannot be claimed
- May NOT be claimed same day as: 03.01AD, 03.01S, 03.01T, 03.03FV, 03.05JR,
  03.08CV, 08.19CV, 08.19CW, or 08.19CX by same physician for same patient
- May NOT be claimed same day as in-person visit or consultation by same physician
  for same patient

Category: V Visit (Virtual)
Base rate: $25.09""",

    'service_context': """This is a BASIC PATIENT-FACING virtual visit by any physician
(not specialist-specific, not physician-to-physician).""",

    'search_criteria': """
WHAT TO LOOK FOR:
- Virtual visits / virtual care
- Telephone consultations / assessments
- Video consultations / assessments
- Telehealth codes
- Any code that can be billed for a patient-facing virtual encounter
""",

    'exclusion_criteria': """
DO NOT INCLUDE:
- Physician-to-physician consultations (e-consults between doctors)
- E-assessments / e-consults (specialist-to-PCP) - not patient-facing
- In-person only codes
- Diagnostic procedures (ECG, imaging, labs)
- Codes you cannot find literally in the text
""",
}
```

#### 4.1.3 Field Usage in Prompts

| Field | Used In | Purpose |
|-------|---------|---------|
| `code` | Phase 1, Output | Displayed as Alberta source code reference |
| `description` | Phase 1, Output | Short description for context |
| `fee` | Phase 1, Output | Fee comparison context |
| `clinical_definition` | Phase 1 | Detailed service definition for matching |
| `service_context` | Phase 1 | Additional LLM guidance |
| `search_criteria` | Phase 1 | Positive matching instructions |
| `exclusion_criteria` | Phase 1 | Negative matching instructions |

### 4.2 PROVINCE_CONFIGS (Cell 5)

#### 4.2.1 Structure

```python
PROVINCE_CONFIGS[prov_code] = {
    'name': str,                    # Full province name
    'chunking_level': int,          # 1 or 2 (section granularity)
    'rules_pages': tuple(int, int), # (start_page, end_page) for preamble
    'skip_sections': list[str],     # Section names to exclude
    'min_clinical_page': int,       # Optional: first clinical content page
    'special_fields': list[str],    # Province-specific output columns
    'extraction_rules': str,        # Province-specific LLM instructions
    'json_schema': dict,            # Expected JSON response structure
    'output_columns': list[str],    # Final DataFrame column order
}
```

#### 4.2.2 British Columbia Configuration

```python
'BC': {
    'name': 'British Columbia',
    'chunking_level': 1,
    'rules_pages': (1, 52),
    'skip_sections': [
        "1. GENERAL PREAMBLE TO THE PAYMENT SCHEDULE",
        "2. OUT-OF-OFFICE HOURS PREMIUMS",
    ],
    'special_fields': [],
    'extraction_rules': """
BC-SPECIFIC EXTRACTION RULES:

1. **CODE PREFIXES** (indicate payment type, NOT setting):
   - P = Professional fee
   - G = Group fee
   - PG = Professional + Group

2. **FEE EXTRACTION**: Copy the exact fee value as shown

ACCURACY RULES - YOU MUST FOLLOW:

1. **ONLY REAL CODES**: Return ONLY codes that LITERALLY appear in the text above.
   - Copy the EXACT code as shown (e.g., 00100, 14051, 97017)
   - If you cannot find the exact code string in the text, DO NOT include it
   - NEVER invent, fabricate, or guess codes

2. **EXACT VALUES**: Copy fee EXACTLY as shown in the document
   - Use exact decimal values (e.g., "25.43" not "25.00")
   - If fee is percentage-based premium, use "-" and explain in condition

3. **FULL DESCRIPTIONS - CLIENT READY FORMAT**:
   - Copy the COMPLETE service description as written in the schedule
   - Do NOT abbreviate (write "Telephone/video consultation" not "Tel consult")
   - Do NOT truncate (include the full description text)
   - Use sentence case for consistency (capitalize first word and proper nouns)
   - Include qualifying details (e.g., "minimum 10 minutes")
   - Format: Clear, professional, ready for client delivery

4. **MODALITY**: Only include modalities explicitly stated
   - "telephone" = text says telephone/phone only
   - "video" = text says video/videoconference only
   - "both" = text explicitly allows BOTH, or doesn't restrict

5. **PAGE NUMBERS**: page_found must match the "=== PAGE X ===" marker where code appears

6. **SECTION HEADING**: Extract the subsection heading the code appears under
   - Look for bold/uppercase headings
   - This becomes level_2_subsection
""",
    'json_schema': {
        'primary_codes': ['code', 'description', 'fee', 'modality', 'page_found',
                         'section_heading', 'reasoning'],
        'add_on_codes': ['code', 'description', 'fee', 'modality', 'page_found',
                        'section_heading', 'links_to', 'condition']
    },
    'output_columns': [
        'AB_Code', 'AB_Description', 'AB_Fee', 'Target_Province',
        'Code', 'Description', 'Fee', 'Type', 'Modality', 'Specialty',
        'Links_To', 'Condition', 'Reasoning',
        'Level_1_Section', 'Level_2_Subsection', 'Page_Found'
    ]
}
```

#### 4.2.3 Manitoba Configuration

```python
'MB': {
    'name': 'Manitoba',
    'chunking_level': 1,
    'rules_pages': (1, 82),
    'skip_sections': ["APPENDICES"],
    'min_clinical_page': 83,  # Clinical sections start at page 83
    'special_fields': [],
    'extraction_rules': """
MB-SPECIFIC EXTRACTION RULES:

1. **SPECIALTY-BASED FEES**: Each specialty section may have its own fee schedules
[... similar accuracy rules ...]
""",
    'json_schema': {
        'primary_codes': ['code', 'description', 'fee', 'modality', 'page_found',
                         'section_heading', 'reasoning'],
        'add_on_codes': ['code', 'description', 'fee', 'modality', 'page_found',
                        'section_heading', 'links_to', 'condition']
    },
    'output_columns': [...]
}
```

#### 4.2.4 Ontario Configuration

```python
'ON': {
    'name': 'Ontario',
    'chunking_level': 2,  # Level 2 for more granular sections
    'rules_pages': (1, 126),
    'skip_sections': [
        "General Preamble",
        "Appendix A", "Appendix B", "Appendix C", "Appendix D",
        "Appendix F", "Appendix G", "Appendix H", "Appendix J", "Appendix Q",
        "Numeric Index",
    ],
    'special_fields': ['Fee_Type', 'Setting', 'Level_3_Heading'],
    'extraction_rules': """
ONTARIO-SPECIFIC EXTRACTION RULES:

1. **H/P COLUMNS (Setting)**:
   - If a code has BOTH H (Hospital) and P (Professional/Office) fees,
     create SEPARATE entries for each
   - H = Hospital setting, P = Professional/Office setting
   - If only one fee exists, use that setting

2. **SURGICAL FEE COLUMNS**:
   - Surg = Surgeon fee -> create entry with fee_type "Surgeon"
   - Asst = Assistant fee -> create entry with fee_type "Assistant" (skip if "nil")
   - Anae = Anaesthesia units -> create entry with fee_type "Anaesthesia"
     (these are TIME UNITS, not dollars)

3. **CODE PREFIXES** (indicate service type, NOT setting):
   - A = Assessments/consultations
   - E = Diagnostic/therapeutic procedures
   - G = General listings
   - K = Special visit premiums
   - Z = Surgical procedures

IMPORTANT: For codes with multiple fee types (Surg/Asst/Anae) or settings (H/P),
create SEPARATE entries for each combination.
[... accuracy rules ...]
""",
    'json_schema': {
        'codes': ['code', 'description', 'fee', 'fee_type', 'setting', 'modality',
                 'page_found', 'level_3_heading', 'is_addon', 'links_to',
                 'condition', 'reasoning']
    },
    'output_columns': [
        'AB_Code', 'AB_Description', 'AB_Fee', 'Target_Province',
        'Code', 'Description', 'Fee', 'Fee_Type', 'Setting', 'Type', 'Modality',
        'Links_To', 'Condition', 'Reasoning',
        'Level_1_Section', 'Level_2_Subsection', 'Level_3_Heading', 'Page_Found'
    ]
}
```

#### 4.2.5 Saskatchewan Configuration

```python
'SK': {
    'name': 'Saskatchewan',
    'chunking_level': 1,
    'rules_pages': (1, 70),
    'skip_sections': [
        "Introduction",
        "To Request a Change to the Payment Schedule",
        "Services Provided Outside Saskatchewan",
        "Billing For Services Provided To Out-Of-Province Beneficiaries",
        "Definitions",
        "Documentation Requirements",
        "Services Billable by Entitlement or by Approval",
        "Assessment Rules",
        "General Information",
        "Services Not Insured by the Ministry of Health",
        "Assessment of Accounts",
        "Verification Program",
        "Information Sources",
        "Reciprocal Billing",
        "Explanatory Codes for Physicians",
    ],
    'min_clinical_page': 71,
    'special_fields': ['Fee_Type', 'Age_Premium_Applies'],
    'extraction_rules': """
SASKATCHEWAN-SPECIFIC EXTRACTION RULES:

1. **DUAL-FEE STRUCTURE (Referred vs Not Referred)**:
   - Many SK codes have TWO fees: "Referred" and "Not Referred"
   - If a code has BOTH fees, create SEPARATE entries for each:
     - One entry with fee_type="Referred" and the referred fee
     - One entry with fee_type="Not Referred" and the not-referred fee
   - If only one fee exists, use fee_type="Standard"

2. **AGE PREMIUMS (Section-Wide)**:
   - SK has age-based premiums for patients 0-5 years and 65+ years
   - If the section header or preamble states age premiums apply to ALL codes
     in the section, note this for EVERY code
   - Do NOT skip age premiums just because they're not repeated per-code

IMPORTANT: For codes with BOTH Referred and Not Referred fees,
create SEPARATE entries for each fee type.
[... accuracy rules ...]
""",
    'json_schema': {
        'primary_codes': ['code', 'description', 'fee', 'fee_type', 'modality',
                         'page_found', 'section_heading', 'age_premium_applies',
                         'reasoning'],
        'add_on_codes': ['code', 'description', 'fee', 'fee_type', 'modality',
                        'page_found', 'section_heading', 'age_premium_applies',
                        'links_to', 'condition']
    },
    'output_columns': [
        'AB_Code', 'AB_Description', 'AB_Fee', 'Target_Province',
        'Code', 'Description', 'Fee', 'Fee_Type', 'Type', 'Modality', 'Specialty',
        'Links_To', 'Condition', 'Reasoning',
        'Level_1_Section', 'Level_2_Subsection', 'Page_Found', 'Age_Premium_Applies'
    ]
}
```

#### 4.2.6 Configuration Summary Table

| Parameter | BC | MB | ON | SK |
|-----------|----|----|----|----|
| **Chunking Level** | 1 | 1 | 2 | 1 |
| **Rules Pages** | 1-52 | 1-82 | 1-126 | 1-70 |
| **Min Clinical Page** | N/A | 83 | N/A | 71 |
| **Skip Sections** | 2 | 1 | 11 | 15 |
| **Special Fields** | None | None | Fee_Type, Setting, Level_3_Heading | Fee_Type, Age_Premium_Applies |
| **JSON Schema Type** | primary/add_on | primary/add_on | codes | primary/add_on |
| **Code Prefix System** | P/G/PG | None | A/E/G/K/Z | None |
| **Fee Structure** | Single | Single | H/P + Surg/Asst/Anae | Referred/Not Referred |
| **Age Premium** | No | No | No | Yes (0-5, 65+) |

---

## 5. Core Functions (Cell 6)

### 5.1 Cost Tracking Functions

#### 5.1.1 reset_cost_tracking()

```python
def reset_cost_tracking():
    """Reset cost tracking for a new province run."""
    global total_cost, total_calls
    total_cost = 0.0
    total_calls = 0
```

Called at the start of each province's processing to reset cumulative cost tracking.

#### 5.1.2 track_cost(inp_tokens, out_tokens)

```python
def track_cost(inp_tokens, out_tokens):
    """Track API costs (GPT-4 pricing estimate)."""
    global total_cost, total_calls
    total_cost += (inp_tokens/1e6)*3.0 + (out_tokens/1e6)*15.0
    total_calls += 1
```

**Pricing Model:**
- Input tokens: $3.00 per 1M tokens
- Output tokens: $15.00 per 1M tokens

Called after every API call with `response.usage.prompt_tokens` and `response.usage.completion_tokens`.

### 5.2 Token Limit Functions

#### 5.2.1 get_dynamic_max_tokens(char_count)

```python
def get_dynamic_max_tokens(char_count):
    """Set max_completion_tokens based on section size for Phase 1."""
    if char_count > 150000:
        return 20000
    elif char_count > 80000:
        return 14000
    elif char_count > 40000:
        return 10000
    elif char_count > 15000:
        return 6000
    else:
        return 4000
```

**Rationale:** Larger sections may contain more matching codes, requiring more output tokens. The thresholds are based on empirical testing to balance cost vs. completeness.

| Section Size | Max Tokens | Typical Use Case |
|--------------|------------|------------------|
| >150K chars | 20,000 | Very large specialty sections (e.g., ON Surgery) |
| >80K chars | 14,000 | Large sections with many codes |
| >40K chars | 10,000 | Medium sections |
| >15K chars | 6,000 | Small-medium sections |
| ≤15K chars | 4,000 | Small sections |

#### 5.2.2 get_phase2_max_tokens(rules_char_count)

```python
def get_phase2_max_tokens(rules_char_count):
    """Set max_completion_tokens for Phase 2 based on rules size."""
    if rules_char_count > 300000:
        return 4000
    elif rules_char_count > 200000:
        return 3000
    else:
        return 2500
```

**Rationale:** Phase 2 output is a fixed JSON schema, so token requirements are more consistent. Larger rules text requires slightly more output buffer for complex attribute extraction.

### 5.3 PDF Processing Functions

#### 5.3.1 load_pdf_pages(pdf_path)

```python
def load_pdf_pages(pdf_path):
    """Load all pages from PDF into a dictionary."""
    pdf_pages = {}
    with pdfplumber.open(pdf_path) as pdf:
        total_pages = len(pdf.pages)
        for i, page in enumerate(tqdm(pdf.pages, desc="Loading pages")):
            page_num = i + 1  # 1-indexed
            try:
                text = page.extract_text()
                if text:
                    pdf_pages[page_num] = text
            except:
                pass  # Skip pages that fail extraction
    return pdf_pages, total_pages
```

**Returns:**
- `pdf_pages`: dict[int, str] - Mapping of page numbers to extracted text
- `total_pages`: int - Total page count for end-page calculations

**Library:** pdfplumber (better for tabular/structured PDFs)

#### 5.3.2 extract_rules_text(pdf_path, rules_pages)

```python
def extract_rules_text(pdf_path, rules_pages):
    """Extract rules/preamble text from PDF."""
    start_page, end_page = rules_pages
    rules_text = ""

    src_pdf = fitz.open(pdf_path)
    for page_num in range(start_page - 1, end_page):  # 0-indexed
        page = src_pdf[page_num]
        text = page.get_text()
        if text:
            rules_text += f"\n=== RULES PAGE {page_num + 1} ===\n{text}"
    src_pdf.close()

    return rules_text
```

**Library:** PyMuPDF/fitz (better for text-heavy extraction)

**Note:** Rules text is extracted with page markers (`=== RULES PAGE X ===`) to help the LLM locate specific rule references.

### 5.4 Section Chunking Functions

#### 5.4.1 build_section_chunks_level1(df_ref, pdf_pages, total_pages, prov_config)

```python
def build_section_chunks_level1(df_ref, pdf_pages, total_pages, prov_config):
    """Build section chunks for Level 1 chunking (BC, MB, SK)."""
    skip_sections = prov_config['skip_sections']
    min_page = prov_config.get('min_clinical_page', 1)

    # Get unique Level 1 sections with their minimum page_start
    level_1_sections = df_ref.groupby('level_1')['page_start'].min().sort_values()
    level_1_list = list(level_1_sections.items())

    section_chunks = {}

    for idx, (section_name, start_page) in enumerate(level_1_list):
        # Skip configured sections
        if section_name in skip_sections:
            continue

        # Skip pages before clinical content
        if start_page < min_page:
            continue

        # End page is start of next section - 1, or last page
        if idx + 1 < len(level_1_list):
            end_page = level_1_list[idx + 1][1] - 1
        else:
            end_page = total_pages

        # Extract text with page markers
        section_text = ""
        pages_in_section = []
        for pg in range(start_page, end_page + 1):
            if pg in pdf_pages:
                section_text += f"\n=== PAGE {pg} ===\n{pdf_pages[pg]}"
                pages_in_section.append(pg)

        section_chunks[section_name] = {
            'text': section_text,
            'level_1': section_name,
            'level_2': section_name,  # Same as level_1 for Level 1 chunking
            'start_page': start_page,
            'end_page': end_page,
            'page_count': len(pages_in_section),
            'char_count': len(section_text)
        }

    return section_chunks
```

**Logic:**
1. Group reference CSV by `level_1` and get minimum `page_start` per section
2. Sort sections by page number
3. For each section:
   - Skip if in `skip_sections` list
   - Skip if before `min_clinical_page`
   - Calculate end page as (next section start - 1) or total pages
   - Concatenate all page texts with `=== PAGE X ===` markers

#### 5.4.2 build_section_chunks_level2(df_ref, pdf_pages, total_pages, prov_config)

```python
def build_section_chunks_level2(df_ref, pdf_pages, total_pages, prov_config):
    """Build section chunks for Level 2 chunking (ON)."""
    skip_sections = prov_config['skip_sections']

    df_ref = df_ref.copy()
    df_ref['level_2'] = df_ref['level_2'].fillna('')

    section_chunks = {}

    for idx, row in df_ref.iterrows():
        level_1 = row['level_1']
        level_2 = row['level_2'] if row['level_2'] else level_1
        start_page = int(row['page_start'])

        if level_1 in skip_sections:
            continue

        # Section key combines level_1 and level_2
        section_key = f"{level_1} | {level_2}" if level_2 != level_1 else level_1

        # End page from next row
        if idx + 1 < len(df_ref):
            end_page = int(df_ref.iloc[idx + 1]['page_start']) - 1
        else:
            end_page = total_pages

        # Build section text
        section_text = ""
        pages_in_section = []
        for pg in range(start_page, end_page + 1):
            if pg in pdf_pages:
                section_text += f"\n=== PAGE {pg} ===\n{pdf_pages[pg]}"
                pages_in_section.append(pg)

        section_chunks[section_key] = {
            'text': section_text,
            'level_1': level_1,
            'level_2': level_2,
            'start_page': start_page,
            'end_page': end_page,
            'page_count': len(pages_in_section),
            'char_count': len(section_text)
        }

    return section_chunks
```

**Key Difference from Level 1:**
- Uses each row in reference CSV as a separate chunk (not grouped by level_1)
- Section key format: `"level_1 | level_2"` for subsections
- Required for Ontario's complex hierarchical structure

### 5.5 Prompt Builder Functions

#### 5.5.1 build_phase1_prompt(section_key, section_info, prov_code, prov_config)

**Full Prompt Structure:**

```
You are a senior physician billing specialist mapping Alberta fee codes to {province} equivalents.

ALBERTA CODE TO MATCH:
- Code: {ab['code']}
- Description: {ab['description']}
- Fee: ${ab['fee']}

CLINICAL SERVICE DEFINITION:
{ab['clinical_definition']}

{ab['service_context']}

You are reviewing the section: {level_1} > {level_2}
Pages {start_page} to {end_page}

{PROVINCE} PAYMENT SCHEDULE - SECTION:

{section_text}  [Full section text with === PAGE X === markers]

TASK:
Find ALL {province} codes in this section that bill for patient-facing virtual assessments
(telephone or video consultations with patients).

{prov_config['extraction_rules']}  [Province-specific extraction rules]

{ab['search_criteria']}  [What to look for]

{ab['exclusion_criteria']}  [What to exclude]

JSON only:
{json_template}  [Province-specific JSON schema]

If no telehealth/virtual codes in this section: {no_match}
```

**Province-Specific JSON Templates:**

**BC/MB:**
```json
{
  "section_name": "{section_key}",
  "found": true,
  "primary_codes": [
    {
      "code": "EXACT code",
      "description": "COMPLETE description",
      "fee": "EXACT fee",
      "modality": "telephone|video|both",
      "page_found": 123,
      "section_heading": "subsection heading",
      "reasoning": "why this matches"
    }
  ],
  "add_on_codes": [
    {
      "code": "EXACT code",
      "description": "COMPLETE description",
      "fee": "EXACT fee",
      "modality": "telephone|video|both",
      "page_found": 123,
      "section_heading": "subsection heading",
      "links_to": ["primary_code"],
      "condition": "when to use"
    }
  ]
}
```

**Ontario:**
```json
{
  "section_key": "{section_key}",
  "found": true,
  "codes": [
    {
      "code": "EXACT code",
      "description": "COMPLETE description",
      "fee": "EXACT fee",
      "fee_type": "Standard|Surgeon|Assistant|Anaesthesia",
      "setting": "Hospital|Professional|N/A",
      "modality": "telephone|video|both",
      "page_found": 123,
      "level_3_heading": "subsection heading",
      "is_addon": false,
      "links_to": [],
      "condition": "",
      "reasoning": "why this matches"
    }
  ]
}
```

**Saskatchewan:**
```json
{
  "section_name": "{section_key}",
  "found": true,
  "primary_codes": [
    {
      "code": "EXACT code",
      "description": "COMPLETE description",
      "fee": "EXACT fee",
      "fee_type": "Referred|Not Referred|Standard",
      "modality": "telephone|video|both",
      "page_found": 123,
      "section_heading": "subsection heading",
      "age_premium_applies": true,
      "reasoning": "why this matches"
    }
  ],
  "add_on_codes": [...]
}
```

#### 5.5.2 build_phase2_prompt(code_info, chunk_text, rules_text, prov_code)

**Full Prompt Structure:**

```
You are a senior physician billing specialist extracting detailed attributes
for a {prov_code} billing code.

CODE TO ANALYZE:
- Code: {code_info['Code']}
- Description: {code_info['Description']}
- Fee: {code_info['Fee']}
- Type: {code_info['Type']}
- Section: {code_info.get('Level_1_Section', 'N/A')}
- Condition (from Phase 1): {code_info.get('Condition', 'N/A')}

ATTRIBUTES TO EXTRACT:
{taxonomy_reference}  [Generated from extraction_taxonomy.xlsx]

{sk_rules}  [SK only: additional rules checklist]

RULES/PREAMBLE (FULL - read carefully for billing rules applicable to this code):
{rules_text}  [FULL - NOT TRUNCATED]

CODE-SPECIFIC SECTION:
{chunk_text[:30000]}  [TRUNCATED to 30K chars]

TASK:
Using ALL available information above, extract values for each attribute.

INSTRUCTIONS:
1. Read through the ENTIRE Rules/Preamble to find billing rules that apply to this code
2. Look for: frequency limits, time requirements, same-day exclusions, premiums, conditions
3. For each attribute, extract the value if found, or null if not explicitly stated
4. For same_day_exclusions: return as array of code strings
5. For additional_notes: ONLY include important billing information not captured elsewhere

Return JSON only:
{
  "modality": "telephone|video|both|in_person|asynchronous|null",
  "minimum_time_minutes": integer or null,
  "frequency_per_day": integer or null,
  "frequency_per_year": integer or null,
  "frequency_per_year_period": "annual|quarterly|90_days|monthly|null",
  "same_day_exclusions": ["code1", "code2"] or [] or null,
  "premium_extended_hours": "rate% code conditions" or null,
  "premium_location": "rate% code conditions" or null,
  "premium_age": "rate% conditions" or null,
  "premium_other": "rate% code conditions" or null,
  "additional_notes": "other important billing info" or null
}
```

**SK-Specific Rules Checklist:**
```
SASKATCHEWAN STANDARD RULES CHECKLIST:
- 3,000 services per physician per year limit
- Age premiums: 0-5 years and 65+ years eligible for premium
- Verify if this code/section is eligible for age premiums
```

### 5.6 Result Processing Function

#### 5.6.1 process_phase1_result(result, section_key, section_info, prov_code, code_chunks)

This function normalizes the province-specific JSON responses into a standardized row format.

**Processing Logic by Province:**

**Ontario (`prov_code == 'ON'`):**
```python
for c in result.get('codes', []):
    unique_key = f"{code}_{fee}_{fee_type}_{setting}_{section_key}"
    code_chunks[unique_key] = section_info['text']

    rows.append({
        'AB_Code': ab['code'],
        'AB_Description': ab['description'],
        'AB_Fee': ab['fee'],
        'Target_Province': prov_code,
        'Code': c.get('code', ''),
        'Description': c.get('description', ''),
        'Fee': c.get('fee', ''),
        'Fee_Type': c.get('fee_type', 'Standard'),
        'Setting': c.get('setting', 'N/A'),
        'Type': 'ADD-ON' if c.get('is_addon') else 'PRIMARY',
        'Modality': c.get('modality', ''),
        'Links_To': ', '.join(c.get('links_to', [])) if c.get('links_to') else '',
        'Condition': c.get('condition', ''),
        'Reasoning': c.get('reasoning', ''),
        'Level_1_Section': section_info.get('level_1', section_key),
        'Level_2_Subsection': section_info.get('level_2', ''),
        'Level_3_Heading': c.get('level_3_heading', ''),
        'Page_Found': c.get('page_found', ''),
        '_unique_key': unique_key
    })
```

**Saskatchewan (`prov_code == 'SK'`):**
- Processes `primary_codes` and `add_on_codes` separately
- Includes `Fee_Type` (Referred/Not Referred/Standard)
- Includes `Age_Premium_Applies` boolean

**BC/MB (default):**
- Processes `primary_codes` and `add_on_codes` separately
- Standard output columns without special fields

**Unique Key Generation:**
- Format varies by province to ensure uniqueness
- Used for Phase 2 merge and chunk text lookup
- Examples:
  - ON: `"{code}_{fee}_{fee_type}_{setting}_{section_key}"`
  - SK: `"{code}_{fee}_{fee_type}_{modality}_{section_key}"`
  - BC/MB: `"{code}_{fee}_{modality}_{section_key}"`

---

## 6. Output Schema

### 6.1 Complete Column Reference

#### 6.1.1 Alberta Source Columns

| Column | Type | Source | Description |
|--------|------|--------|-------------|
| `AB_Code` | string | ALBERTA_CODE_CONFIG | Alberta HSC code (e.g., "03.03CV") |
| `AB_Description` | string | ALBERTA_CODE_CONFIG | Alberta code description |
| `AB_Fee` | float | ALBERTA_CODE_CONFIG | Alberta base fee amount |

#### 6.1.2 Target Province Columns

| Column | Type | Source | Description |
|--------|------|--------|-------------|
| `Target_Province` | string | Processing | Province code: BC, MB, ON, SK |
| `Code` | string | Phase 1 | Extracted provincial billing code |
| `Description` | string | Phase 1 | Full code description from schedule |
| `Fee` | string | Phase 1 | Fee amount (may include currency, modifiers) |
| `Type` | string | Phase 1 | PRIMARY (standalone) or ADD-ON (modifier) |
| `Modality` | string | Phase 1 | telephone, video, both, or empty |
| `Specialty` | string | Phase 1 | Section name (for BC/MB/SK) |
| `Links_To` | string | Phase 1 | Comma-separated related codes (for add-ons) |
| `Condition` | string | Phase 1 | Billing conditions/restrictions |
| `Reasoning` | string | Phase 1 | LLM reasoning for why code matches |

#### 6.1.3 Location Columns

| Column | Type | Source | Description |
|--------|------|--------|-------------|
| `Level_1_Section` | string | Phase 1 | Top-level section name |
| `Level_2_Subsection` | string | Phase 1 | Second-level subsection |
| `Level_3_Heading` | string | Phase 1 (ON) | Third-level heading (Ontario only) |
| `Page_Found` | integer | Phase 1 | PDF page number where code appears |

#### 6.1.4 Province-Specific Columns

**Ontario:**
| Column | Type | Values | Description |
|--------|------|--------|-------------|
| `Fee_Type` | string | Standard, Surgeon, Assistant, Anaesthesia | Type of fee from H/P or surgical columns |
| `Setting` | string | Hospital, Professional, N/A | H (Hospital) or P (Professional/Office) setting |

**Saskatchewan:**
| Column | Type | Values | Description |
|--------|------|--------|-------------|
| `Fee_Type` | string | Referred, Not Referred, Standard | Referral status affecting fee |
| `Age_Premium_Applies` | boolean | true/false | Whether age premium (0-5, 65+) applies |

#### 6.1.5 Phase 2 Attribute Columns

| Column | Type | Description |
|--------|------|-------------|
| `modality` | string | Mode: telephone, video, both, in_person, asynchronous, null |
| `minimum_time_minutes` | integer | Minimum physician time required (e.g., 10) |
| `frequency_per_day` | integer | Maximum claims per patient per day |
| `frequency_per_year` | integer | Maximum claims per patient per year |
| `frequency_per_year_period` | string | Period type: annual, quarterly, 90_days, monthly |
| `same_day_exclusions` | string | Comma-separated codes that cannot be billed same day |
| `premium_extended_hours` | string | Extended hours premium (rate%, code, conditions) |
| `premium_location` | string | Location premium (rate%, code, conditions) |
| `premium_age` | string | Age premium (rate%, conditions) |
| `premium_other` | string | Other premiums (rate%, code, conditions) |
| `additional_notes` | string | Other important billing information |

---

## 7. API Configuration

### 7.1 Model Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `model` | `gpt-4.1-2025-04-14` | Latest GPT-4.1 for accurate extraction |
| `temperature` | 0.1 | Low temperature for consistent, deterministic output |
| `max_completion_tokens` | Dynamic | See Token Management section |

### 7.2 API Call Pattern

```python
resp = client.chat.completions.create(
    model="gpt-4.1-2025-04-14",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.1,
    max_completion_tokens=max_tokens
)

# Track costs
track_cost(resp.usage.prompt_tokens, resp.usage.completion_tokens)

# Extract response
content = resp.choices[0].message.content

# Parse JSON from response
match = re.search(r'\{[\s\S]*\}', content)
if match:
    result = json.loads(match.group())
```

### 7.3 Error Handling

```python
try:
    resp = client.chat.completions.create(...)
    # Process response
except Exception as e:
    print(f"  [{section_key[:40]}] ERROR: {e}")
    # For Phase 2: append empty result with unique_key
    phase2_results.append({'_unique_key': unique_key})
```

---

## 8. Dependencies

### 8.1 Python Packages

```python
# Core
openai>=1.0.0          # OpenAI API client (ChatCompletion interface)
pandas>=2.0.0          # DataFrame operations, Excel I/O

# PDF Processing
pdfplumber>=0.9.0      # PDF text extraction (page loading, tables)
PyMuPDF>=1.22.0        # PDF text extraction (rules/preamble)

# Excel Output
openpyxl>=3.1.0        # Excel file writing (.xlsx)

# Progress Display
tqdm>=4.65.0           # Progress bars for loops
```

### 8.2 Installation Command

```bash
pip install openai pandas pdfplumber openpyxl tqdm PyMuPDF -q
```

### 8.3 Google Colab Imports

```python
from google.colab import files  # File upload/download
from tqdm.notebook import tqdm  # Notebook-compatible progress bars
```

---

## 9. File Structure

```
project/
├── 3.03 All Province_Alberta Crosswalk.ipynb        # Main production notebook
├── 3.03 All Province_Alberta Crosswalk_Truncated.ipynb  # Fast testing version
├── README_Crosswalk_Documentation.md                # This documentation
│
├── extraction_taxonomy.xlsx                         # Phase 2 attribute definitions
│
├── bc_section_reference_simple.csv                  # BC PDF section index
├── manitoba_section_reference_final.csv             # MB PDF section index
├── on_section_reference_full.csv                    # ON PDF section index
├── sk_section_reference_simple.csv                  # SK PDF section index
│
├── resources/
│   └── schedules/
│       ├── BC_Physician_Manual.pdf                  # BC Payment Schedule
│       ├── MB_Physician_Manual.pdf                  # MB Physician's Manual
│       ├── ON_Schedule_of_Benefits_2024.pdf         # ON Schedule of Benefits
│       └── SK_Fee_Schedule_2025.pdf                 # SK Payment Schedule
│
└── output/
    ├── 3.02_BC_Alberta_Complete.xlsx                # BC crosswalk results
    ├── 3.02_MB_Alberta_Complete.xlsx                # MB crosswalk results
    ├── 3.02_ON_Alberta_Complete.xlsx                # ON crosswalk results
    ├── 3.02_SK_Alberta_Complete.xlsx                # SK crosswalk results
    └── 3.03_All_Province_03_03CV_Complete.xlsx      # Combined results
```

---

## 10. Execution Reference

### 10.1 Cell Execution Order

| Cell | Name | Purpose | Dependencies |
|------|------|---------|--------------|
| 1 | Install Dependencies | Install pip packages | None |
| 2a | Upload Province PDFs | Load PDF files | Cell 1 |
| 2b | Upload Section Reference CSVs | Load section indices | Cell 1 |
| 2c | Upload Extraction Taxonomy | Load attribute definitions | Cell 1 |
| 3 | API Key | Initialize OpenAI client | Cell 1 |
| 4 | Alberta Code Config | Configure source code | Cell 1 |
| 5 | Province Configs | Load province configurations | Cell 1 |
| 6 | Shared Functions | Define processing functions | Cells 1, 4, 5 |
| 7a | Run British Columbia | Process BC | Cells 2a-6 |
| 7b | Run Manitoba | Process MB | Cells 2a-6 |
| 7c | Run Ontario | Process ON | Cells 2a-6 |
| 7d | Run Saskatchewan | Process SK | Cells 2a-6 |
| 8 | Combine & Summary | Merge all provinces | Cells 7a-7d |

### 10.2 Typical Processing Time

| Province | Sections | Codes | Phase 1 | Phase 2 | Total |
|----------|----------|-------|---------|---------|-------|
| BC | ~30 | 10-30 | 5-10 min | 3-8 min | 8-18 min |
| MB | ~40 | 15-40 | 8-15 min | 5-12 min | 13-27 min |
| ON | ~80 | 30-80 | 15-30 min | 10-25 min | 25-55 min |
| SK | ~25 | 10-25 | 4-8 min | 3-7 min | 7-15 min |

### 10.3 Typical API Costs

| Province | API Calls | Estimated Cost |
|----------|-----------|----------------|
| BC | 40-60 | $2-5 |
| MB | 55-80 | $3-7 |
| ON | 110-160 | $8-15 |
| SK | 35-50 | $2-4 |
| **Total** | **240-350** | **$15-31** |

---

## 11. Known Limitations

### 11.1 Alberta Code Configuration
The current implementation has the Alberta code hardcoded in Cell 4. For production use, this should be refactored to read from an external master file containing all Alberta billing codes with their complete metadata.

### 11.2 Section Reference Dependencies
Each provincial PDF requires a manually-created section reference CSV. This indexing process is labor-intensive and must be updated when PDFs are revised. Each provincial PDF should be indexed into a master section reference sheet to improve extraction efficiency and efficacy.

### 11.3 Extraction Taxonomy
The attributes defined in `extraction_taxonomy.xlsx` require a complete overhaul:
- Current attribute definitions do not adequately capture the complexity of provincial billing rules
- Inconsistent attribute granularity across provinces
- Missing province-specific attributes
- Ambiguous definitions leading to extraction inconsistencies
- Lack of validation rules for extracted values
- No support for conditional attributes (e.g., premiums that only apply to certain code types)

### 11.4 Phase 1 TASK Line
The Phase 1 prompt contains a hardcoded TASK description ("patient-facing virtual assessments"). For non-telehealth codes, a `task_description` field must be added to `ALBERTA_CODE_CONFIG` and the prompt updated to use `{ab['task_description']}`.

### 11.5 PDF Text Extraction
- Tables may not extract correctly depending on PDF formatting
- Multi-column layouts can produce interleaved text
- Some PDFs have OCR quality issues affecting extraction accuracy

### 11.6 LLM Limitations
- May hallucinate codes that don't exist in the text
- Inconsistent fee extraction when formats vary
- Description truncation for very long descriptions
- May miss codes when sections are very large (>150K chars)

---

## 12. Version History

| Version | Date | Changes |
|---------|------|---------|
| 3.03 | 2024 | Consolidated all-province notebook with unified configuration |
| 3.02 | 2024 | Individual province notebooks with province-specific optimizations |
| 3.01 | 2024 | Initial two-phase extraction implementation |
| 2.x | 2024 | Single-phase extraction (deprecated) |
| 1.x | 2023 | Manual crosswalk process (deprecated) |
