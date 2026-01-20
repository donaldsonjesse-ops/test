# Universal Billing Code Extraction - Complete Build Guide

**Project:** Ontario Schedule of Benefits Extraction  
**Purpose:** Extract 190-column billing profiles from MOH fee schedule with government-grade accuracy  
**Author:** HelpSeeker Technologies  
**Date:** January 2025

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Code Audit](#2-current-code-audit)
3. [Architecture Overview](#3-architecture-overview)
4. [Rule Cache Reference](#4-rule-cache-reference)
5. [Complete Notebook Code](#5-complete-notebook-code)
6. [Quality Assurance](#6-quality-assurance)
7. [Testing Protocol](#7-testing-protocol)

---

## 1. Executive Summary

### What This Extraction Does

Takes a crosswalk file (mapping Alberta codes to Ontario equivalents) and extracts a full 190-column billing profile for each Ontario code from the MOH Schedule of Benefits PDF.

### Critical Problems in Current Code

| Issue | Impact | Fix |
|-------|--------|-----|
| Multi-rate codes not split | Wrong fees for codes like E079 | Your crosswalk already handles this—two rows per multi-rate code |
| Time premium defaults wrong | Claims `DEFAULT_YES` but GP65 restricts eligibility | Apply GP65-78 rules programmatically |
| Age premium guessing | LLM infers instead of checking GP64 list | Hard-code the eligible code list |
| E-codes misclassified | Premium codes treated as services | Add classification step, set service__ fields to N/A |
| Referral rules inconsistent | Should follow GP16-20 | Consultations always require referral |

### Accuracy Target

| Field Category | Current | Target |
|----------------|---------|--------|
| Explicit values (YES/NO from text) | ~31% | 50%+ |
| Rule-based values (GP rules) | Unknown | 95%+ |
| UNKNOWN values | ~5% | <2% |
| Multi-rate handling | Broken | 100% |

---

## 2. Current Code Audit

### Test Results (3 codes: A101, A102, E079)

**A101 - Limited Virtual Care by Video ($20.00)**
- ✅ Base rate correct
- ✅ Telehealth video = YES, audio = NO
- ❌ `time_premium__after_hours_*` = DEFAULT_YES (should check GP65)
- ❌ `setting__office` = NO (correct) but other settings DEFAULT_YES without validation

**A102 - Limited Virtual Care by Telephone ($15.00)**
- ✅ Base rate correct
- ✅ Telehealth audio = YES, video = NO
- ❌ Same time premium issue
- ❌ Several UNKNOWN values in rules

**E079 - Smoking Cessation Premium**
- ❌ Only captured $15.55, missed $13.20 telephone rate (but your crosswalk fixes this)
- ❌ `service__consultation` = YES (wrong - this is a premium, not a service)
- ✅ `links_to_primary` = A101, A102 (correct)

### Value Distribution Analysis

From test extraction:
```
DEFAULT_NO:  107 values
DEFAULT_YES:  76 values  ← Too many inferences
YES:          31 values
NO:           58 values
UNKNOWN:      14 values
CONDITIONAL:  12 values
```

**Problem:** 183 DEFAULT values vs 89 explicit values means the LLM is guessing, not extracting.

---

## 3. Architecture Overview

### Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           INPUT                                          │
├─────────────────────────────────────────────────────────────────────────┤
│  Crosswalk Excel          Ontario Schedule PDF                          │
│  - Code (column E)        - 978 pages                                   │
│  - Description            - General Preamble (GP1-GP110)                │
│  - Fee                    - Code listings                               │
│  - Type (PRIMARY/ADD-ON)                                                │
│  - Modality                                                             │
│  - Links_To                                                             │
│  - Pages (column O)                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        PRE-PROCESSING                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  1. Load Rule Cache (GP64 age premiums, GP65-78 time premiums, etc.)   │
│  2. Load PDF pages into memory                                          │
│  3. Filter crosswalk to Target_Province = 'ON'                         │
│  4. Rename columns to match expected format                             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      PER-CODE EXTRACTION                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ CALL 0: Classification (API)                                     │   │
│  │ - Determine code type (VIRTUAL_CARE, CONSULTATION, etc.)        │   │
│  │ - Check if specialist-only                                       │   │
│  │ - Identify linked codes                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ CALLS 1-5: Field Extraction (API)                                │   │
│  │ - Extract ONLY explicitly stated values                          │   │
│  │ - Use NOT_FOUND instead of guessing                              │   │
│  │ - Pass classification context to guide extraction                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ RULE APPLICATION (Python - No API)                               │   │
│  │ - Apply GP64 age premium rules                                   │   │
│  │ - Apply GP65-78 time premium eligibility                         │   │
│  │ - Apply GP16-20 referral requirements                            │   │
│  │ - Set N/A for incompatible field categories                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            OUTPUT                                        │
├─────────────────────────────────────────────────────────────────────────┤
│  Excel file: 190 columns per row                                        │
│  - May have more rows than input (if multi-rate codes in crosswalk)    │
│  - QA metrics for review                                                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Value System (8 values)

| Value | When to Use | Example |
|-------|-------------|---------|
| `YES` | Explicitly stated in schedule text | "payable in any setting" |
| `NO` | Explicitly prohibited in schedule | "not eligible for payment with..." |
| `DEFAULT_YES` | Logically required, silence = permission | `jurisdiction__in_province` for OHIP |
| `DEFAULT_NO` | Logically incompatible with code type | `setting__office` for virtual-only code |
| `N/A` | Field category doesn't apply | `procedure__bilateral` for phone consultation |
| `CONDITIONAL:{reason}` | Depends on circumstances | "may apply if patient is inpatient" |
| `RULE_BASED:{rule}` | Set by GP rule application | Age premiums per GP64 |
| `NOT_FOUND` | Could not locate in context | Needs human review |

---

## 4. Rule Cache Reference

These rules come from the General Preamble and MUST be applied programmatically, not by LLM inference.

### GP64: Age-Based Fee Premiums (Page 78)

**15% Premium for Age 65+ - Eligible Codes:**
```python
AGE_PREMIUM_65_PLUS_CODES = [
    # General assessments
    'A003', 'C003', 'C903', 'W102', 'W109', 'W903',
    # Re-assessments  
    'A004', 'C004', 'W004',
    # Intermediate assessment
    'A007',
    # Focused practice assessments
    'A917', 'A927', 'A937', 'A947', 'A957', 'A967',
    # Periodic health visit 65+
    'K132',
    # Comprehensive assessment and care
    'H102', 'H122', 'H132', 'H152',
    # Multiple systems assessment
    'H103', 'H123', 'H133', 'H153'
]
```

**Pediatric Premiums - Apply to:**
- Specialist consultations (limited, repeat)
- Surgical procedures (Parts K-Z)
- Surgical assistant services (Parts K-Z)
- Clinical procedures with diagnostic radiology
- Ophthalmology specific assessments (A233, A234)

**Pediatric Rates:**
| Age Range | Premium |
|-----------|---------|
| 0-29 days | 30% |
| 30 days to < 1 year | 25% |
| 1 to < 2 years | 20% |
| 2 to < 5 years | 15% |
| 5 to < 16 years | 10% |

### GP65-78: Special Visit Premium Rules (Pages 79-92)

**NOT Eligible for Special Visit Premiums:**
- Patients seen during rounds at hospital or LTC
- Admission assessments for elective hospital admission
- Non-referred or transferred obstetrical patients (except C989)
- Services in scheduled open facilities (clinics, urgent care, walk-in)
- When critical care team fees are payable
- Sleep study services
- Patients presenting without appointment during/around office hours
- **H-prefix emergency department codes**

**Eligible Sections:**
- Consultations and Visits
- Diagnostic and Therapeutic Procedures

**Time Definitions:**
| Period | Hours |
|--------|-------|
| Daytime | 07:00-17:00 Mon-Fri |
| Evening | 17:00-24:00 Mon-Fri |
| Night | 00:00-07:00 |
| Weekend | Sat-Sun 07:00-24:00 |

### GP16-20: Consultation Rules (Pages 30-34)

**Consultation Requirements:**
- Written referral required from physician, NP, or dental surgeon
- Written report required to referring provider
- Frequency limit: 1 per same diagnosis per 24 months

**Repeat Consultation Requirements:**
- New written request from referring provider
- Care rendered by another physician in interval since first consultation

### Code Classification Patterns

```python
CODE_TYPE_PATTERNS = {
    'VIRTUAL_CARE': {
        'codes': ['A101', 'A102'],
        'patterns': [r'A1[0-9]{2}', r'.*[Vv]ideo.*', r'.*[Tt]elephone.*'],
        'forced_values': {
            'setting__office': 'NO',
            'setting__telehealth_synchronous': 'YES',
            'setting__virtual_care': 'YES'
        }
    },
    'PREMIUM_ADDON': {
        'patterns': [r'E0[0-9]{2}', r'E4[0-9]{2}'],
        'forced_values': {
            'all service__ fields': 'N/A',
            'all complexity_level__ fields': 'N/A'
        },
        'requires': 'links_to_primary must be populated'
    },
    'CONSULTATION': {
        'indicators': ['consultation', 'consult'],
        'forced_values': {
            'rule__referral_required': 'YES',
            'rule__written_report_required': 'YES',
            'referral__referred': 'YES'
        }
    },
    'ASSESSMENT_GP': {
        'codes': ['A001', 'A003', 'A004', 'A007', 'A008'],
        'forced_values': {
            'rule__referral_required': 'NO',
            'referral__referral_not_required': 'YES',
            'all procedure__ fields': 'N/A'
        }
    },
    'SURGICAL': {
        'prefixes': ['K', 'L', 'M', 'N', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z'],
        'note': 'Only when in surgical sections, not all K-codes',
        'age_premiums': True
    }
}
```

---

## 5. Complete Notebook Code

### Cell 1: Title and Documentation

```python
# =============================================================================
# UNIVERSAL BILLING CODE EXTRACTION
# =============================================================================
# 
# Pipeline Step 2:
#   1. Crosswalk notebook finds applicable codes → exports results
#   2. THIS NOTEBOOK extracts full 190-column profile for each code
#
# Output: Wide table (1 row per code/modality, 190 columns)
#
# Value System (8 values):
#   YES           - Explicitly stated as allowed/applicable in schedule
#   NO            - Explicitly stated as prohibited/not applicable
#   DEFAULT_YES   - Not stated, but logically required by code type
#   DEFAULT_NO    - Not stated, but logically incompatible with code type
#   N/A           - Field category doesn't apply to this service type
#   CONDITIONAL   - Depends on circumstances (includes reason)
#   RULE_BASED    - Set by General Preamble rule application
#   NOT_FOUND     - Could not locate in context, needs human review
#
# Key Improvement: Age/time premiums set by GP rules, not LLM inference
# =============================================================================
```

### Cell 2: Setup

```python
!pip install openai pandas pdfplumber openpyxl tqdm -q
print('Dependencies installed')
```

### Cell 3: Column Definitions (190 columns - UNCHANGED)

```python
# Fixed identifier columns (first 6)
ID_COLUMNS = [
    'billing_code',
    'province',
    'billing_name',
    'base_rate',
    'modality',
    'source_document'
]

# All 184 field columns organized by category
FIELD_COLUMNS = [
    # Core identification
    'core__billing_code_variant',
    'core__fee_schedule_name',
    'core__fee_schedule_version',
    'core__section_code',
    'core__category_code',
    'core__effective_date',

    # Code structure
    'structure__code_formula',
    'structure__code_prefix_meaning',
    'structure__code_suffix_meaning',
    'structure__code_section_range',
    'structure__modifier_approach',
    'structure__code_family',
    'structure__in_office_variant',
    'structure__out_of_office_variant',
    'structure__virtual_variant',
    'structure__provincial_code_format',

    # Specialty
    'specialty__specialty_code',
    'specialty__specialty_name',
    'specialty__specialty_section',
    'specialty__specialty_subsection',
    'specialty__specialty_is_gp',
    'specialty__specialty_modifier_required',
    'specialty__specialty_rate_tier',
    'specialty__specialty_certification_required',

    # Care setting
    'setting__office',
    'setting__out_of_office',
    'setting__home',
    'setting__hospital_inpatient',
    'setting__hospital_outpatient',
    'setting__emergency_department',
    'setting__urgent_care',
    'setting__nursing_home',
    'setting__long_term_care',
    'setting__telehealth_synchronous',
    'setting__telehealth_asynchronous',
    'setting__virtual_care',
    'setting__ambulatory_surgical_center',
    'setting__community_health_center',
    'setting__correctional_facility',
    'setting__mobile_clinic',

    # Jurisdiction
    'jurisdiction__in_province',
    'jurisdiction__out_of_province_canadian',
    'jurisdiction__out_of_country',
    'jurisdiction__reciprocal_billing',

    # Age modifiers
    'age__age_0_1',
    'age__age_under_2',
    'age__age_2_12',
    'age__age_under_13',
    'age__age_13_17',
    'age__age_18_49',
    'age__age_50_59',
    'age__age_60_64',
    'age__age_65_69',
    'age__age_70_79',
    'age__age_80_plus',
    'age__age_not_applicable',
    'age__age_band_type',
    'age__age_modifier_rate',
    'age__age_modifier_code',

    # Time premiums
    'time_premium__after_hours_evening',
    'time_premium__after_hours_night',
    'time_premium__after_hours_weekend',
    'time_premium__after_hours_holiday',
    'time_premium__emergency_premium',
    'time_premium__on_call_premium',

    # Callback
    'callback__callback_evening',
    'callback__callback_night',
    'callback__callback_weekend',
    'callback__callback_holiday',
    'callback__unscheduled_service',

    # Telehealth
    'telehealth__telehealth_video',
    'telehealth__telehealth_audio',
    'telehealth__telehealth_store_forward',
    'telehealth__virtual_care_standard',
    'telehealth__virtual_care_comprehensive',
    'telehealth__e_consult',

    # Complexity
    'complexity__complexity_low',
    'complexity__complexity_moderate',
    'complexity__complexity_high',
    'complexity__complexity_extended',
    'complexity__acuity_modifier',

    # Location modifiers
    'location__hospital_modifier',
    'location__facility_fee',
    'location__rural_modifier',
    'location__remote_modifier',
    'location__northern_modifier',

    # Procedure modifiers
    'procedure__bilateral_modifier',
    'procedure__multiple_procedure',
    'procedure__repeat_procedure',
    'procedure__surgical_assist',
    'procedure__second_surgeon',
    'procedure__anesthesia_modifier',

    # Bundling
    'bundling__bundling_approach',
    'bundling__bundled_with_codes',
    'bundling__unbundled_components',
    'bundling__technical_component',
    'bundling__professional_component',
    'bundling__global_service',
    'bundling__component_billing_allowed',
    'bundling__bundle_exception_rules',

    # Billing rules
    'rule__frequency_limit',
    'rule__frequency_period',
    'rule__referral_required',
    'rule__approval_required',
    'rule__certification_required',
    'rule__written_report_required',
    'rule__same_day_restriction',
    'rule__same_provider_restriction',
    'rule__location_restriction',
    'rule__time_documentation_required',
    'rule__diagnostic_restriction',

    # Duration
    'duration__time_unit',
    'duration__minimum_time',
    'duration__maximum_time',
    'duration__time_rounding_rule',
    'duration__concurrent_billing_allowed',
    'duration__cumulative_time_allowed',

    # Frequency limits
    'frequency__frequency_per_day',
    'frequency__frequency_per_week',
    'frequency__frequency_per_month',
    'frequency__frequency_per_year',
    'frequency__frequency_per_lifetime',
    'frequency__frequency_per_episode',
    'frequency__frequency_exception_criteria',

    # Service type classification
    'service__visit_assessment',
    'service__visit_comprehensive',
    'service__consultation',
    'service__hospital_visit',
    'service__emergency_visit',
    'service__home_visit',
    'service__diagnostic_test',
    'service__diagnostic_imaging',
    'service__laboratory',
    'service__surgical_procedure',
    'service__therapeutic_procedure',
    'service__counselling',
    'service__preventive_care',
    'service__chronic_disease_management',

    # Complexity level
    'complexity_level__brief',
    'complexity_level__limited',
    'complexity_level__intermediate',
    'complexity_level__comprehensive',
    'complexity_level__complex',
    'complexity_level__extended',
    'complexity_level__prolonged',

    # Referral status
    'referral__referred',
    'referral__non_referred',
    'referral__self_referred',
    'referral__referral_not_required',
    'referral__internal_referral',
    'referral__external_referral',

    # Rate calculation
    'rate__base_rate',
    'rate__rate_currency',
    'rate__rate_type',
    'rate__rate_unit',
    'rate__rate_base_for_percentage',
    'rate__rate_cap',
    'rate__rate_floor',
    'rate__max_with_specialty',
    'rate__max_with_time_premium',
    'rate__max_with_all_modifiers',
    'rate__pediatric_rate',
    'rate__geriatric_rate',
    'rate__rural_rate',
    'rate__rate_effective_date',
    'rate__rate_expiry_date',
    'rate__rate_previous',
    'rate__rate_change_percent',

    # Metadata
    'links_to_primary',
    'source_pages',
    'source_section',
    'extraction_notes'
]

ALL_COLUMNS = ID_COLUMNS + FIELD_COLUMNS
print(f"Total columns: {len(ALL_COLUMNS)}")
print(f"  - ID columns: {len(ID_COLUMNS)}")
print(f"  - Field columns: {len(FIELD_COLUMNS)}")
```

### Cell 4: Rule Cache (NEW - Critical for Accuracy)

```python
# =============================================================================
# RULE CACHE - From General Preamble (GP1-GP110)
# These rules are DETERMINISTIC - apply programmatically, not via LLM
# =============================================================================

# GP64: Age-Based Fee Premiums (15% for 65+)
AGE_PREMIUM_65_PLUS_CODES = [
    'A003', 'C003', 'C903', 'W102', 'W109', 'W903',  # General assessments
    'A004', 'C004', 'W004',                          # Re-assessments
    'A007',                                          # Intermediate assessment
    'A917', 'A927', 'A937', 'A947', 'A957', 'A967',  # Focused practice
    'K132',                                          # Periodic health visit 65+
    'H102', 'H122', 'H132', 'H152',                  # Comprehensive assessment
    'H103', 'H123', 'H133', 'H153'                   # Multiple systems
]

# GP64: Pediatric premium rates (for surgical, specialist consultations)
PEDIATRIC_PREMIUM_RATES = {
    '0-29_days': 0.30,
    '30d-1yr': 0.25,
    '1-2yr': 0.20,
    '2-5yr': 0.15,
    '5-16yr': 0.10
}

# GP65-78: Special visit premium eligibility
# These code types are ELIGIBLE for special visit premiums
SPECIAL_VISIT_ELIGIBLE_TYPES = ['ASSESSMENT_GP', 'ASSESSMENT_SPECIALIST', 'CONSULTATION', 'DIAGNOSTIC']

# These are explicitly NOT eligible per GP65
SPECIAL_VISIT_EXCLUDED = [
    'H-prefix codes (emergency dept on-duty)',
    'Sleep study codes',
    'Critical care team fees'
]

# GP16-20: Consultation requirements
CONSULTATION_RULES = {
    'referral_required': True,
    'written_report_required': True,
    'frequency_same_diagnosis': '1 per 24 months'
}

# Code classification patterns
VIRTUAL_CARE_CODES = ['A101', 'A102', 'A101A', 'A102A']
PREMIUM_ADDON_PATTERN = r'^E[0-9]{3}$'  # E-prefix 3-digit codes in premium tables
GP_ASSESSMENT_CODES = ['A001', 'A003', 'A004', 'A007', 'A008']

# Surgical section prefixes (for age premium eligibility)
SURGICAL_PREFIXES = ['K', 'L', 'M', 'N', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z']

print("Rule cache loaded:")
print(f"  - {len(AGE_PREMIUM_65_PLUS_CODES)} codes eligible for 65+ age premium")
print(f"  - {len(VIRTUAL_CARE_CODES)} virtual care codes")
print(f"  - {len(GP_ASSESSMENT_CODES)} GP assessment codes")
```

### Cell 5: Upload Files

```python
from google.colab import files
import pandas as pd

print("Upload the following files:")
print("  1. Crosswalk results (xlsx) - with Ontario codes")
print("  2. Ontario PDF (Schedule of Benefits)")
print()
print("Select ALL files (Ctrl+click):")
uploaded = files.upload()

CROSSWALK_FILE = ONTARIO_PDF = None
for f in uploaded.keys():
    if f.endswith('.xlsx'):
        CROSSWALK_FILE = f
    elif f.endswith('.pdf'):
        ONTARIO_PDF = f

print(f"\nCrosswalk: {CROSSWALK_FILE}")
print(f"PDF: {ONTARIO_PDF}")

# Load crosswalk and filter to Ontario
raw_df = pd.read_excel(CROSSWALK_FILE)
print(f"\nRaw crosswalk rows: {len(raw_df)}")

# Filter to Ontario only (adjust column name if different)
if 'Target_Province' in raw_df.columns:
    crosswalk_df = raw_df[raw_df['Target_Province'] == 'ON'].copy()
else:
    crosswalk_df = raw_df.copy()  # Assume all Ontario if no province column

# Rename columns to match extraction code expectations
column_mapping = {
    'Code': 'ON_Code',
    'Description': 'ON_Description',
    'Fee': 'ON_Fee',
    'Type': 'Type',
    'Modality': 'Modality',
    'Links_To': 'Links_To',
    'Pages': 'Pages'
}
for old_col, new_col in column_mapping.items():
    if old_col in crosswalk_df.columns:
        crosswalk_df = crosswalk_df.rename(columns={old_col: new_col})

print(f"\nOntario codes to process: {len(crosswalk_df)}")
print("\nPreview:")
display_cols = ['ON_Code', 'ON_Description', 'ON_Fee', 'Type', 'Modality', 'Pages']
display_cols = [c for c in display_cols if c in crosswalk_df.columns]
print(crosswalk_df[display_cols].to_string())
```

### Cell 6: API Key

```python
OPENAI_API_KEY = ""  # <-- Paste your key here

if not OPENAI_API_KEY:
    from getpass import getpass
    OPENAI_API_KEY = getpass("OpenAI API Key: ")

from openai import OpenAI
client = OpenAI(api_key=OPENAI_API_KEY)
print("API ready")
```

### Cell 7: Load PDF

```python
import pdfplumber
from tqdm.notebook import tqdm

print("Loading PDF...")
pdf_pages = {}
with pdfplumber.open(ONTARIO_PDF) as pdf:
    for i, page in enumerate(tqdm(pdf.pages, desc="Loading pages")):
        try:
            text = page.extract_text()
            if text:
                pdf_pages[i + 1] = text
        except:
            pass

print(f"PDF pages loaded: {len(pdf_pages)}")

# Also identify General Preamble pages for reference
gp_pages = []
for p, text in pdf_pages.items():
    if 'GENERAL PREAMBLE' in text.upper() or (p <= 110 and 'GP' in text):
        gp_pages.append(p)
print(f"General Preamble pages identified: {len(gp_pages)} pages")
```

### Cell 8: Extraction Logic (MAJOR CHANGES)

```python
import json
import re

# =============================================================================
# COST TRACKING
# =============================================================================
total_cost = 0.0
total_calls = 0

def track_cost(inp, out):
    """Track API costs (adjust rates for your model)."""
    global total_cost, total_calls
    # GPT-4 pricing - adjust if using different model
    total_cost += (inp/1e6)*30.0 + (out/1e6)*60.0
    total_calls += 1

# =============================================================================
# PAGE FINDING
# =============================================================================
def find_code_pages(code, pages_hint=None):
    """Find PDF pages containing this code."""
    found_pages = []
    
    # Use hint if provided (e.g., "191-200")
    if pages_hint and pages_hint not in ['nan', 'None', '']:
        try:
            parts = str(pages_hint).replace(' ', '').split('-')
            start = max(1, int(parts[0]) - 2)  # Small buffer
            end = min(max(pdf_pages.keys()), int(parts[-1]) + 2)
            for p in range(start, end + 1):
                if p in pdf_pages and code in pdf_pages[p]:
                    found_pages.append(p)
        except:
            pass
    
    # Fallback: search all pages
    if not found_pages:
        for p, text in pdf_pages.items():
            if code in text:
                found_pages.append(p)
    
    return sorted(found_pages)[:10]  # Limit to 10 pages max

# =============================================================================
# VALUE RULES (Stricter than before)
# =============================================================================
VALUE_RULES = """
VALUE ASSIGNMENT RULES - STRICT VERSION:

1. YES = Schedule EXPLICITLY states this is allowed/applicable
   → You MUST be able to quote the source text
   
2. NO = Schedule EXPLICITLY prohibits or excludes this
   → You MUST be able to quote the source text
   
3. DEFAULT_YES = Not stated, but LOGICALLY REQUIRED by code type
   → ONLY use when omission would be illogical
   → Example: jurisdiction__in_province for any OHIP code
   
4. DEFAULT_NO = Not stated and LOGICALLY INCOMPATIBLE with code type
   → ONLY use when code type makes this impossible
   → Example: setting__office for a virtual-only code
   
5. N/A = Field category DOES NOT APPLY to this service type
   → Example: procedure__bilateral for a phone consultation
   
6. CONDITIONAL:{reason} = Depends on specific circumstances in schedule
   
7. NOT_FOUND = Could not locate explicit information in provided context
   → Use this instead of guessing - it will be reviewed

CRITICAL: DO NOT USE DEFAULT_YES OR DEFAULT_NO FOR:
- Age premiums (will be set by GP64 rules)
- Time premiums (will be set by GP65-78 rules)  
- Referral requirements (will be set by GP16-20 rules)

These fields will be populated by rule application, not LLM inference.
"""

# =============================================================================
# CLASSIFICATION FUNCTION
# =============================================================================
def classify_code(code, description, fee, context):
    """Classify code type before extraction."""
    
    prompt = f"""Classify Ontario billing code {code}.

CODE: {code}
DESCRIPTION: {description}
FEE: ${fee}

SCHEDULE CONTENT:
{context[:8000]}

Determine the PRIMARY TYPE. Choose exactly ONE:
- VIRTUAL_CARE: Telehealth/video/telephone services (A101, A102, etc.)
- PREMIUM_ADDON: Premium codes that add to base services (E-prefix in premium tables)
- CONSULTATION: Consultation services requiring referral
- ASSESSMENT_GP: General practice assessments (A001, A003, A004, A007, A008)
- ASSESSMENT_SPECIALIST: Specialist assessments
- DIAGNOSTIC: Diagnostic tests/imaging
- SURGICAL: Surgical procedures (K-Z prefix in surgical sections)
- THERAPEUTIC: Therapeutic procedures
- MANAGEMENT_FEE: Monthly/annual management fees

Also determine:
- Is this SPECIALIST_ONLY? (requires specialist credentials)
- What SECTION is this from?

Return ONLY this JSON:
{{
    "primary_type": "...",
    "specialist_only": true/false,
    "section": "...",
    "subsection": "..."
}}"""

    try:
        resp = client.chat.completions.create(
            model="gpt-4o-2024-11-20",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
            max_tokens=300
        )
        track_cost(resp.usage.prompt_tokens, resp.usage.completion_tokens)
        
        content = resp.choices[0].message.content
        match = re.search(r'\{[\s\S]*\}', content)
        if match:
            result = json.loads(match.group())
            return result
    except Exception as e:
        print(f"    Classification error: {e}")
    
    # Fallback classification based on code pattern
    if code in VIRTUAL_CARE_CODES:
        return {"primary_type": "VIRTUAL_CARE", "specialist_only": False, "section": "Virtual Care", "subsection": ""}
    elif re.match(PREMIUM_ADDON_PATTERN, code):
        return {"primary_type": "PREMIUM_ADDON", "specialist_only": False, "section": "Premiums", "subsection": ""}
    elif code in GP_ASSESSMENT_CODES:
        return {"primary_type": "ASSESSMENT_GP", "specialist_only": False, "section": "Family Practice", "subsection": ""}
    else:
        return {"primary_type": "UNKNOWN", "specialist_only": False, "section": "", "subsection": ""}

# =============================================================================
# RULE APPLICATION (No API - Deterministic)
# =============================================================================
def apply_gp_rules(row, classification):
    """Apply General Preamble rules programmatically."""
    
    code = row['billing_code']
    primary_type = classification.get('primary_type', 'UNKNOWN')
    
    # -------------------------------------------------------------------------
    # GP64: Age Premium Rules
    # -------------------------------------------------------------------------
    if code in AGE_PREMIUM_65_PLUS_CODES:
        row['age__age_65_plus'] = 'YES'
        row['age__age_modifier_rate'] = '15%'
        row['age__age_not_applicable'] = 'NO'
        row['age__age_band_type'] = 'geriatric_65_plus'
    elif primary_type == 'SURGICAL' or (code[0] in SURGICAL_PREFIXES and primary_type not in ['PREMIUM_ADDON', 'MANAGEMENT_FEE']):
        # Surgical codes get pediatric premiums
        row['age__age_0_1'] = 'YES'
        row['age__age_under_2'] = 'YES' 
        row['age__age_2_12'] = 'YES'
        row['age__age_13_17'] = 'YES'
        row['age__age_not_applicable'] = 'NO'
        row['age__age_band_type'] = 'pediatric_surgical'
        row['age__age_modifier_rate'] = 'See GP64 table'
    else:
        # Most codes don't have age modifiers
        row['age__age_not_applicable'] = 'YES'
        for f in [c for c in FIELD_COLUMNS if c.startswith('age__') and c != 'age__age_not_applicable' and c != 'age__age_band_type']:
            if row.get(f) in ['NOT_FOUND', 'UNKNOWN', None]:
                row[f] = 'N/A'
    
    # -------------------------------------------------------------------------
    # GP65-78: Special Visit Premium Rules
    # -------------------------------------------------------------------------
    if primary_type in SPECIAL_VISIT_ELIGIBLE_TYPES:
        if not code.startswith('H'):  # H-prefix excluded per GP65
            for f in [c for c in FIELD_COLUMNS if c.startswith('time_premium__')]:
                if row.get(f) in ['NOT_FOUND', 'UNKNOWN', None]:
                    row[f] = 'ELIGIBLE_PER_GP65'
            for f in [c for c in FIELD_COLUMNS if c.startswith('callback__')]:
                if row.get(f) in ['NOT_FOUND', 'UNKNOWN', None]:
                    row[f] = 'ELIGIBLE_PER_GP65'
        else:
            for f in [c for c in FIELD_COLUMNS if c.startswith('time_premium__')]:
                if row.get(f) in ['NOT_FOUND', 'UNKNOWN', None]:
                    row[f] = 'NO_H_PREFIX_EXCLUDED'
            for f in [c for c in FIELD_COLUMNS if c.startswith('callback__')]:
                if row.get(f) in ['NOT_FOUND', 'UNKNOWN', None]:
                    row[f] = 'NO_H_PREFIX_EXCLUDED'
    elif primary_type in ['VIRTUAL_CARE']:
        # Virtual care - special visit depends on circumstances
        for f in [c for c in FIELD_COLUMNS if c.startswith('time_premium__')]:
            if row.get(f) in ['NOT_FOUND', 'UNKNOWN', None]:
                row[f] = 'CONDITIONAL:see_GP65_virtual_rules'
    elif primary_type == 'PREMIUM_ADDON':
        # Premium codes - eligibility follows base service
        for f in [c for c in FIELD_COLUMNS if c.startswith('time_premium__')]:
            if row.get(f) in ['NOT_FOUND', 'UNKNOWN', None]:
                row[f] = 'SEE_BASE_SERVICE'
    
    # -------------------------------------------------------------------------
    # GP16-20: Referral Rules
    # -------------------------------------------------------------------------
    if primary_type == 'CONSULTATION' or 'consult' in row.get('billing_name', '').lower():
        row['rule__referral_required'] = 'YES'
        row['rule__written_report_required'] = 'YES'
        row['referral__referred'] = 'YES'
        row['referral__referral_not_required'] = 'NO'
    elif primary_type == 'ASSESSMENT_GP':
        row['rule__referral_required'] = 'NO'
        row['referral__referral_not_required'] = 'YES'
        row['referral__referred'] = 'NO'
    elif classification.get('specialist_only'):
        row['rule__referral_required'] = 'CONDITIONAL:specialist_rules'
    
    # -------------------------------------------------------------------------
    # Procedure modifiers - N/A for non-procedures
    # -------------------------------------------------------------------------
    if primary_type not in ['SURGICAL', 'THERAPEUTIC', 'DIAGNOSTIC']:
        for f in [c for c in FIELD_COLUMNS if c.startswith('procedure__')]:
            row[f] = 'N/A'
    
    # -------------------------------------------------------------------------
    # Premium add-ons - service fields are N/A
    # -------------------------------------------------------------------------
    if primary_type == 'PREMIUM_ADDON':
        for f in [c for c in FIELD_COLUMNS if c.startswith('service__')]:
            row[f] = 'N/A'
        for f in [c for c in FIELD_COLUMNS if c.startswith('complexity_level__')]:
            row[f] = 'N/A'
        # Verify links_to_primary is set
        if not row.get('links_to_primary') or str(row.get('links_to_primary')) in ['', 'nan', 'None']:
            row['extraction_notes'] = row.get('extraction_notes', '') + ' WARNING:premium_code_missing_links_to_primary'
    
    # -------------------------------------------------------------------------
    # Virtual care - office setting forced NO
    # -------------------------------------------------------------------------
    if primary_type == 'VIRTUAL_CARE':
        row['setting__office'] = 'NO'
        row['setting__telehealth_synchronous'] = 'YES'
        row['setting__virtual_care'] = 'YES'
    
    return row

# =============================================================================
# EXTRACTION CALLS (5 focused calls)
# =============================================================================
EXTRACTION_CALLS = [
    {
        "name": "Core + Structure + Specialty",
        "fields": [
            "core__billing_code_variant", "core__fee_schedule_name", "core__fee_schedule_version",
            "core__section_code", "core__category_code", "core__effective_date",
            "structure__code_formula", "structure__code_prefix_meaning", "structure__code_suffix_meaning",
            "structure__code_section_range", "structure__modifier_approach", "structure__code_family",
            "structure__in_office_variant", "structure__out_of_office_variant", "structure__virtual_variant",
            "structure__provincial_code_format",
            "specialty__specialty_code", "specialty__specialty_name", "specialty__specialty_section",
            "specialty__specialty_subsection", "specialty__specialty_is_gp", "specialty__specialty_modifier_required",
            "specialty__specialty_rate_tier", "specialty__specialty_certification_required"
        ],
        "decision_logic": """
EXTRACT ONLY EXPLICIT VALUES FROM TEXT:
- fee_schedule_name: "Ontario Schedule of Benefits"
- fee_schedule_version: From page header (e.g., "Amd 12 Draft 1")
- effective_date: From page header
- section_code: The section header text exactly as written
- code_family: Related codes listed in same table/section

Use NOT_FOUND if information is not visible in the provided text.
Do NOT infer or guess."""
    },
    {
        "name": "Settings + Telehealth + Jurisdiction",
        "fields": [
            "setting__office", "setting__out_of_office", "setting__home",
            "setting__hospital_inpatient", "setting__hospital_outpatient", "setting__emergency_department",
            "setting__urgent_care", "setting__nursing_home", "setting__long_term_care",
            "setting__telehealth_synchronous", "setting__telehealth_asynchronous", "setting__virtual_care",
            "setting__ambulatory_surgical_center", "setting__community_health_center",
            "setting__correctional_facility", "setting__mobile_clinic",
            "telehealth__telehealth_video", "telehealth__telehealth_audio", "telehealth__telehealth_store_forward",
            "telehealth__virtual_care_standard", "telehealth__virtual_care_comprehensive", "telehealth__e_consult",
            "jurisdiction__in_province", "jurisdiction__out_of_province_canadian",
            "jurisdiction__out_of_country", "jurisdiction__reciprocal_billing"
        ],
        "decision_logic": """
SETTING RULES:
- If "by Video" in code name: telehealth__telehealth_video = YES, telehealth__telehealth_audio = NO
- If "by Telephone" in code name: telehealth__telehealth_video = NO, telehealth__telehealth_audio = YES
- If "video or telephone" mentioned: both = YES
- If "payable in any setting" stated: applicable settings = YES
- If specific setting mentioned: that setting = YES

JURISDICTION:
- jurisdiction__in_province = DEFAULT_YES (OHIP requirement)
- Others = NOT_FOUND unless explicitly stated

Use CONDITIONAL:{reason} when text says "may apply if..."
"""
    },
    {
        "name": "Rules + Bundling",
        "fields": [
            "rule__frequency_limit", "rule__frequency_period",
            "rule__approval_required", "rule__certification_required",
            "rule__same_day_restriction", "rule__same_provider_restriction", "rule__location_restriction",
            "rule__time_documentation_required", "rule__diagnostic_restriction",
            "bundling__bundling_approach", "bundling__bundled_with_codes", "bundling__unbundled_components",
            "bundling__technical_component", "bundling__professional_component", "bundling__global_service",
            "bundling__component_billing_allowed", "bundling__bundle_exception_rules"
        ],
        "decision_logic": """
EXTRACT ONLY EXPLICIT STATEMENTS:

Look for these specific phrases:
- "not payable same day as..." → rule__same_day_restriction = "CONDITIONAL:{codes listed}"
- "maximum X per..." → rule__frequency_limit = X, rule__frequency_period = period
- "start and stop times must be recorded" → rule__time_documentation_required = YES
- "prior approval required" → rule__approval_required = YES
- "includes..." or "bundled with..." → bundling__bundled_with_codes = [codes]
- "not eligible for payment with..." → bundling rules

DO NOT FILL: rule__referral_required, rule__written_report_required (set by GP rules)
If not explicitly stated → NOT_FOUND"""
    },
    {
        "name": "Duration + Rates + Frequency",
        "fields": [
            "duration__time_unit", "duration__minimum_time", "duration__maximum_time",
            "duration__time_rounding_rule", "duration__concurrent_billing_allowed", "duration__cumulative_time_allowed",
            "frequency__frequency_per_day", "frequency__frequency_per_week", "frequency__frequency_per_month",
            "frequency__frequency_per_year", "frequency__frequency_per_lifetime", "frequency__frequency_per_episode",
            "frequency__frequency_exception_criteria",
            "rate__base_rate", "rate__rate_currency", "rate__rate_type", "rate__rate_unit",
            "rate__rate_base_for_percentage", "rate__rate_cap", "rate__rate_floor",
            "rate__rate_effective_date", "rate__rate_expiry_date", "rate__rate_previous", "rate__rate_change_percent"
        ],
        "decision_logic": """
DURATION - Extract if time requirements stated:
- "minimum of X minutes" → duration__minimum_time = X
- "per 15 minutes" → duration__time_unit = "15 minutes", rate__rate_type = "time-based"
- If flat fee with no time mention → duration fields = N/A

FREQUENCY - Extract explicit limits only:
- "maximum one per..." → frequency__frequency_per_{period}
- If no limit stated → NOT_FOUND (do NOT assume "no limit")

RATES:
- rate__base_rate = the fee amount
- rate__rate_currency = "CAD"
- rate__rate_type = "flat" for fixed, "percentage" if % of another code, "time-based" if per unit
- rate__rate_unit = "per service" unless otherwise specified

DO NOT FILL: rate__pediatric_rate, rate__geriatric_rate (set by GP64 rules)"""
    },
    {
        "name": "Service Type + Modifiers",
        "fields": [
            "service__visit_assessment", "service__visit_comprehensive", "service__consultation",
            "service__hospital_visit", "service__emergency_visit", "service__home_visit",
            "service__diagnostic_test", "service__diagnostic_imaging", "service__laboratory",
            "service__surgical_procedure", "service__therapeutic_procedure", "service__counselling",
            "service__preventive_care", "service__chronic_disease_management",
            "complexity_level__brief", "complexity_level__limited", "complexity_level__intermediate",
            "complexity_level__comprehensive", "complexity_level__complex", "complexity_level__extended",
            "complexity_level__prolonged",
            "complexity__complexity_low", "complexity__complexity_moderate",
            "complexity__complexity_high", "complexity__complexity_extended", "complexity__acuity_modifier",
            "location__hospital_modifier", "location__facility_fee", "location__rural_modifier",
            "location__remote_modifier", "location__northern_modifier"
        ],
        "decision_logic": """
SERVICE TYPE - Based on code description:
- If "assessment" in name → service__visit_assessment = YES
- If "consultation" in name → service__consultation = YES
- If "Limited" in name → complexity_level__limited = YES, others = NO
- If no complexity term → all complexity_level fields = N/A

LOCATION MODIFIERS - Extract only if explicitly mentioned in schedule.

NOTE: If this is a PREMIUM_ADDON code, all service__ fields should be N/A.
      Procedure modifiers handled by rule application."""
    }
]

# =============================================================================
# MAKE API CALL
# =============================================================================
def make_focused_call(code, description, fee, context, call_config, classification):
    """Make a single focused extraction call."""
    
    fields_list = "\n".join([f'  "{f}": "value",' for f in call_config["fields"]])
    
    prompt = f"""Extract billing information for Ontario code {code}.

CODE: {code}
DESCRIPTION: {description}
FEE: ${fee}
CLASSIFICATION: {classification.get('primary_type', 'UNKNOWN')}

SCHEDULE CONTENT:
{context}

{VALUE_RULES}

{call_config["decision_logic"]}

Return JSON with these fields:
{{
{fields_list}
}}

Use NOT_FOUND rather than guessing. Some fields will be set by rule application."""

    try:
        resp = client.chat.completions.create(
            model="gpt-4o-2024-11-20",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.1,
            max_tokens=2000
        )
        track_cost(resp.usage.prompt_tokens, resp.usage.completion_tokens)
        
        content = resp.choices[0].message.content
        match = re.search(r'\{[\s\S]*\}', content)
        
        if match:
            return json.loads(match.group())
    except Exception as e:
        print(f"    Error in {call_config['name']}: {e}")
    
    return {}

# =============================================================================
# MAIN EXTRACTION FUNCTION
# =============================================================================
def extract_code_profile(code, description, fee, modality, code_type, pages_hint, links_to):
    """Extract full profile: Classification → 5 Extraction Calls → Rule Application."""
    
    code_pages = find_code_pages(code, pages_hint)
    
    # Initialize row
    row = {
        'billing_code': code,
        'province': 'ON',
        'billing_name': description,
        'base_rate': fee,
        'modality': modality,
        'source_document': 'Ontario Schedule of Benefits 2024'
    }
    
    # Initialize all fields to NOT_FOUND
    for col in FIELD_COLUMNS:
        row[col] = 'NOT_FOUND'
    
    if not code_pages:
        print(f"    Warning: No pages found for {code}")
        row['extraction_notes'] = 'INSUFFICIENT_DATA: No pages found'
        row['links_to_primary'] = links_to
        return row
    
    context = "\n".join([f"=== PAGE {p} ===\n{pdf_pages[p]}" for p in code_pages])
    
    # -------------------------------------------------------------------------
    # STEP 1: Classification (API Call 0)
    # -------------------------------------------------------------------------
    print(f"    Call 0/6: Classification")
    classification = classify_code(code, description, fee, context)
    print(f"      → Type: {classification.get('primary_type')}")
    
    # -------------------------------------------------------------------------
    # STEP 2: Extraction Calls 1-5
    # -------------------------------------------------------------------------
    for i, call_config in enumerate(EXTRACTION_CALLS):
        print(f"    Call {i+1}/6: {call_config['name']}")
        
        extracted = make_focused_call(code, description, fee, context, call_config, classification)
        
        for field, value in extracted.items():
            if field in row and value not in [None, '', 'null']:
                row[field] = value
    
    # -------------------------------------------------------------------------
    # STEP 3: Apply GP Rules (No API)
    # -------------------------------------------------------------------------
    print(f"    Applying GP rules...")
    row = apply_gp_rules(row, classification)
    
    # -------------------------------------------------------------------------
    # STEP 4: Set metadata
    # -------------------------------------------------------------------------
    row['links_to_primary'] = links_to if links_to and str(links_to) not in ['', 'nan', 'None'] else ''
    row['source_pages'] = str(code_pages)
    row['source_section'] = classification.get('section', '')
    
    # Add classification to notes for QA
    row['extraction_notes'] = (row.get('extraction_notes', '') + 
        f" CLASSIFIED_AS:{classification.get('primary_type')}").strip()
    
    return row

print("="*70)
print("EXTRACTION READY")
print("="*70)
print(f"Calls per code: 6 (1 classification + 5 extraction)")
print(f"Rule application: GP64 (age), GP65-78 (time), GP16-20 (referral)")
print(f"Fields per extraction call: {[len(c['fields']) for c in EXTRACTION_CALLS]}")
print("="*70)
```

### Cell 9: Run Extraction

```python
print("="*70)
print("EXTRACTING FULL PROFILES")
print("="*70)
print(f"Codes to process: {len(crosswalk_df)}")
print(f"Expected API calls: {len(crosswalk_df) * 6}")
print()

all_rows = []

for i, row in tqdm(crosswalk_df.iterrows(), total=len(crosswalk_df), desc="Extracting"):
    code = row.get('ON_Code', '')
    desc = row.get('ON_Description', '')
    fee = row.get('ON_Fee', 0)
    modality = row.get('Modality', 'both')
    code_type = row.get('Type', 'PRIMARY')
    pages_hint = str(row.get('Pages', ''))
    links_to = row.get('Links_To', '') if code_type == 'ADD-ON' else ''
    
    print(f"\n[{i+1}/{len(crosswalk_df)}] {code} - {desc[:50]}...")
    
    profile = extract_code_profile(
        code=code,
        description=desc,
        fee=fee,
        modality=modality,
        code_type=code_type,
        pages_hint=pages_hint,
        links_to=links_to
    )
    
    all_rows.append(profile)

print("\n" + "="*70)
print("EXTRACTION COMPLETE")
print("="*70)
print(f"Total rows: {len(all_rows)}")
print(f"Total API calls: {total_calls}")
print(f"Estimated cost: ${total_cost:.2f}")
print("="*70)
```

### Cell 10: Save Results

```python
# Create DataFrame with all columns in order
result_df = pd.DataFrame(all_rows, columns=ALL_COLUMNS)

# Save to Excel
output_file = 'universal_billing_extraction.xlsx'
result_df.to_excel(output_file, index=False)
print(f"Saved {len(result_df)} rows x {len(result_df.columns)} columns to {output_file}")

from google.colab import files
files.download(output_file)
```

### Cell 11: Preview

```python
# Show key columns for preview
preview_cols = [
    'billing_code', 'billing_name', 'base_rate', 'modality',
    'setting__office', 'setting__virtual_care',
    'telehealth__telehealth_video', 'telehealth__telehealth_audio',
    'age__age_not_applicable', 'age__age_65_plus',
    'rule__referral_required',
    'links_to_primary', 'extraction_notes'
]
preview_cols = [c for c in preview_cols if c in result_df.columns]

print("Preview of key fields:")
print(result_df[preview_cols].to_string())
```

### Cell 12: QA Summary (NEW)

```python
print("="*70)
print("QUALITY ASSURANCE SUMMARY")
print("="*70)

# Count value types across all field columns
from collections import Counter

value_counts = Counter()
for col in FIELD_COLUMNS:
    if col in result_df.columns:
        for val in result_df[col].astype(str):
            # Normalize value type
            if ':' in val:
                val_type = val.split(':')[0]
            else:
                val_type = val
            value_counts[val_type] += 1

print("\n1. VALUE DISTRIBUTION:")
print("-" * 40)
for val, count in value_counts.most_common(15):
    pct = count / sum(value_counts.values()) * 100
    print(f"   {val:25} {count:6} ({pct:5.1f}%)")

# Check for issues
print("\n2. POTENTIAL ISSUES:")
print("-" * 40)

# NOT_FOUND count
not_found = value_counts.get('NOT_FOUND', 0)
print(f"   NOT_FOUND values: {not_found}")
if not_found > 0:
    # Which fields have most NOT_FOUND?
    nf_by_field = {}
    for col in FIELD_COLUMNS:
        if col in result_df.columns:
            nf_count = (result_df[col] == 'NOT_FOUND').sum()
            if nf_count > 0:
                nf_by_field[col] = nf_count
    print("   Top NOT_FOUND fields:")
    for field, count in sorted(nf_by_field.items(), key=lambda x: -x[1])[:5]:
        print(f"      {field}: {count}")

# Check premium codes have links
print("\n3. PREMIUM CODE VALIDATION:")
print("-" * 40)
premium_mask = result_df['billing_code'].str.match(r'^E[0-9]{3}$', na=False)
premium_codes = result_df[premium_mask]
if len(premium_codes) > 0:
    missing_links = premium_codes[
        (premium_codes['links_to_primary'].isna()) | 
        (premium_codes['links_to_primary'].astype(str).isin(['', 'nan', 'None']))
    ]
    print(f"   Premium codes found: {len(premium_codes)}")
    print(f"   Missing links_to_primary: {len(missing_links)}")
    if len(missing_links) > 0:
        print(f"   Codes: {missing_links['billing_code'].tolist()}")
else:
    print("   No E-prefix premium codes found")

# Classification distribution
print("\n4. CLASSIFICATION DISTRIBUTION:")
print("-" * 40)
if 'extraction_notes' in result_df.columns:
    classifications = result_df['extraction_notes'].str.extract(r'CLASSIFIED_AS:(\w+)')[0]
    for cls, count in classifications.value_counts().items():
        print(f"   {cls}: {count}")

# Rule application check
print("\n5. RULE APPLICATION VERIFICATION:")
print("-" * 40)
# Check age premiums on eligible codes
age_premium_applied = result_df[result_df['billing_code'].isin(AGE_PREMIUM_65_PLUS_CODES)]
if len(age_premium_applied) > 0:
    correct = (age_premium_applied['age__age_65_plus'] == 'YES').sum()
    print(f"   65+ age premium codes in data: {len(age_premium_applied)}")
    print(f"   Correctly marked YES: {correct}")

# Check virtual care settings
virtual_codes = result_df[result_df['billing_code'].isin(VIRTUAL_CARE_CODES)]
if len(virtual_codes) > 0:
    office_no = (virtual_codes['setting__office'] == 'NO').sum()
    virtual_yes = (virtual_codes['setting__virtual_care'] == 'YES').sum()
    print(f"   Virtual care codes in data: {len(virtual_codes)}")
    print(f"   setting__office = NO: {office_no}")
    print(f"   setting__virtual_care = YES: {virtual_yes}")

print("\n" + "="*70)
print("QA COMPLETE")
print("="*70)
```

---

## 6. Quality Assurance

### Manual Verification Checklist

For each extraction run, verify a sample against the source PDF:

| Check | How to Verify |
|-------|---------------|
| Base rate matches | Compare extracted `base_rate` to PDF listing |
| Modality correct | Video code shows video=YES, audio=NO |
| Age premiums correct | Codes in GP64 list show 65+ premium |
| Time premiums correct | Non-H codes show ELIGIBLE, H codes show EXCLUDED |
| Referral rules correct | Consultations show referral_required=YES |
| Premium codes linked | E-codes have links_to_primary populated |
| Section captured | source_section matches PDF header |

### Value Distribution Targets

| Value Type | Target % | Concern If |
|------------|----------|------------|
| YES/NO (explicit) | >40% | <30% means over-inference |
| RULE_BASED/ELIGIBLE_PER | >20% | <10% means rules not applying |
| DEFAULT_YES/NO | <20% | >30% means too much guessing |
| NOT_FOUND | <10% | >15% needs investigation |
| N/A | 15-25% | Very high/low suggests classification issues |

---

## 7. Testing Protocol

### Phase 1: Smoke Test (3 codes)

Run on A101, A102, E079 only. Verify:
- [ ] A101 video rate = $20.00
- [ ] A102 telephone rate = $15.00
- [ ] A101/A102 setting__office = NO
- [ ] A101/A102 setting__virtual_care = YES
- [ ] A101 telehealth_video = YES, telehealth_audio = NO
- [ ] A102 telehealth_video = NO, telehealth_audio = YES
- [ ] E079 service__ fields = N/A
- [ ] E079 links_to_primary populated

### Phase 2: Expanded Test (20 codes)

Add codes from different categories:
- GP assessments (A003, A004)
- Specialist consultations
- Surgical codes (if in crosswalk)
- More premium codes

### Phase 3: Full Run

Run on complete crosswalk. Review QA summary for anomalies.

---

## Appendix: Source References

| Rule | GP Page | Content |
|------|---------|---------|
| Age premiums | GP64 (p.78) | 15% for 65+ on listed codes, pediatric rates |
| Special visits | GP65-78 (p.79-92) | Eligibility criteria, exclusions |
| Consultations | GP16-20 (p.30-34) | Referral and report requirements |
| Assessments | GP15, GP21-39 | Specific elements, types |
| Bundling | GP11-14 | Component billing rules |

---

*Document Version: 1.0*  
*Last Updated: January 2025*  
*Source: Ontario Schedule of Benefits, Amendment 12 Draft 1 (effective March 3, 2025)*
