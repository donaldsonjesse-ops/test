# AB/BC Telehealth Crosswalk Audit

**Source:** BC MSC Payment Schedule (March 31, 2024)  
**Crosswalk:** bc_crosswalk_complete.xlsx (39 rows)

---

## Summary

| Metric | Count |
|--------|-------|
| Total Mappings | 39 |
| Verified Correct | 26 |
| Critical Errors | 5 |
| Moderate Issues | 8 |

---

## Critical Errors

These codes do not exist or are fundamentally misidentified.

### 1. Codes 98010 & 98020 — DO NOT EXIST

**Crosswalk claims:**
- 98010: Telephone/video visit - family physician, patient-initiated, 10 min+, $25.00
- 98020: Telephone/video visit - specialist (non-GP), patient-initiated, 10 min+, $25.00

**Reality:** Neither code exists anywhere in the 508-page BC Payment Schedule. Searched entire document — zero matches.

**Impact:** Rows 0, 1 in crosswalk are invalid.

---

### 2. Code 00040 — WRONG CODE

**Crosswalk claims:**
- Row 34
- Description: "Out-of-office hours premium for virtual care (evenings, weekends, holidays)"
- Links_To: 98010, 98020
- Fee: Listed as "-" (add-on)

**Reality (Page 59):**
```
00040  Stomach lavage and gavage ................................................................................ 26.71
```

This is a gastrointestinal procedure, not a virtual care premium.

**Impact:** Row 34 is completely wrong.

---

### 3. Code 13037 (without P prefix) — WRONG CODE/FEE

**Crosswalk claims (Row 5):**
- Code: 13037
- Description: "Telehealth visit (video) – GP/FP – office/outpatient – per 15 minutes or greater portion thereof"
- Fee: $20.00
- Page_Found: 142

**Reality (Page 114):**
```
P13037  Telehealth GP in-office Visit ................................................................................. 34.44
```

**Errors:**
1. Missing "P" prefix
2. Wrong fee ($20.00 vs $34.44)
3. Wrong description (not per-15-minute billing)
4. Wrong page (142 vs 114)

---

### 4. Code 13018 (without P prefix) — WRONG CODE/FEE

**Crosswalk claims (Row 6):**
- Code: 13018
- Description: "Telehealth visit (video) – GP/FP – home visit – per 15 minutes or greater portion thereof"
- Fee: $20.00
- Page_Found: 142

**Reality (Page 115):**
```
P13018  Telehealth GP out-of-office Individual counselling for a prolonged visit for 
        counselling (minimum time per visit – 20 minutes)............................................... 75.32
```

**Errors:**
1. Missing "P" prefix
2. Wrong fee ($20.00 vs $75.32)
3. Wrong description (it's counselling, not a basic visit)
4. Wrong minimum time (20 min, not 15 min)

---

### 5. Code G14076 — DUPLICATE WITH CONFLICTING FEES

**Crosswalk has two entries:**
- Row 4: G14076, Fee: $15.00
- Row 7: G14076, Fee: $21.92

**Reality (Page 152):**
```
G14076 FP Patient Telephone Management Fee .......................................................................... 21.92
```

**Error:** Row 4 has wrong fee ($15.00). Correct fee is $21.92.

---

## Moderate Issues

### 6. Code PG33250 — MODALITY ERROR

**Crosswalk claims (Rows 16 & 17):**
- Row 16: modality = "telephone"
- Row 17: modality = "video"

**Reality (Page 232):**
```
PG33250 Virtual communication with patient, or representative/family, for medically
        pertinent matters ................................................................................................... 10.96
Notes:  
ii)  Payable for communication with patient or representative/family through
     telephone or email.
```

**Error:** Video is NOT an approved modality. Only telephone or email.

---

### 7. Pediatric Surcharges 50571, 50572, 50573 — LINKAGE ERROR

**Crosswalk claims (Rows 36-38):**
- Links_To: 50507, 50517 (telehealth codes)

**Reality (Page 335):**
```
Notes:
i)   Restricted to Pediatrics and Pediatric Cardiology.
ii)  Payable only in addition to fee items 00510, 00550, 00551, 00585, 01511,
     01512, and 01513.
```

**Error:** The surcharges link to in-person codes (00510, 00550, etc.), NOT to telehealth codes (50507, 50517).

---

### 8. Codes 50572 & 50573 — MODALITY MISMATCH

**Crosswalk claims (Rows 37-38):**
- modality: "in_person"

**Issue:** These are listed in a telehealth crosswalk but marked as in_person modality, which is contradictory to the purpose of the crosswalk.

---

### 9. Code 32377 — MISSING RESTRICTIONS

**Crosswalk (Row 12):**
- Code and fee ($81.00) are correct

**Missing from crosswalk (Page 214):**
```
Notes:
i)   Payable only for General Internal Medicine specialists who have completed 3
     years of core Internal Medicine training plus at least 1 year of General
     Internal Medicine training.
ii)  Payable only if 00311 or 32271 paid within the previous 6 months.
iii) Payable for patients that have 3 or more of the conditions listed...
```

---

## Error Patterns

### Pattern 1: Code Prefix Confusion
BC uses prefixes to denote fee categories:
- `P` = GPSC fees
- `PG` = Specific program fees
- `G` = GPSC incentive fees

**Affected rows:** 5, 6 (13037→P13037, 13018→P13018)

### Pattern 2: Fabricated Generic Codes
BC does NOT have universal virtual visit codes. The crosswalk invented 98010/98020 as "catch-all" codes.

**Affected rows:** 0, 1, 34

### Pattern 3: Modality Assumptions
Assumed telephone and video are interchangeable when BC explicitly restricts some codes.

**Affected rows:** 16, 17 (PG33250)

### Pattern 4: Unverified Linkages
Surcharge linkages were assumed rather than verified against source.

**Affected rows:** 36, 37, 38

---

## Verified Correct Codes

| Row | BC Code | Fee | Specialty | Page | Status |
|-----|---------|-----|-----------|------|--------|
| 2 | P13037 | $34.44 | Family Medicine | 114 | ✓ |
| 3 | P13017 | $41.10 | Family Medicine | 115 | ✓ |
| 7 | G14076 | $21.92 | Family Medicine | 152 | ✓ |
| 8 | G14023 | $21.91 | Family Medicine | 158 | ✓ |
| 9 | 20207 | $34.76 | Dermatology | 181 | ✓ |
| 10 | 22007 | $37.46 | Ophthalmology | 188 | ✓ |
| 11 | 32277 | $54.65 | Internal Medicine | 211 | ✓ |
| 12 | 32377 | $81.00 | Gen Internal Med | 214 | ✓ (missing restrictions) |
| 13 | 33107 | $79.44 | Cardiology | 219 | ✓ |
| 14 | 30077 | $40.53 | Immunology/Allergy | 230 | ✓ |
| 15 | 30078 | $22.78 | Immunology/Allergy | 230 | ✓ |
| 16 | PG33250 | $10.96 | Endocrinology | 232 | ✓ (modality error) |
| 18 | 00477 | $97.12 | Neurology | 271 | ✓ |
| 19 | 03317 | $62.78 | Neurosurgery | 276 | ✓ |
| 20 | 50507 | $97.58 | Pediatrics | 335 | ✓ |
| 21 | 50517 | $113.01 | Pediatrics | 335 | ✓ |
| 22 | 60607 | $59.86 | Psychiatry | 348 | ✓ |
| 23 | 01777 | $111.65 | Phys Med & Rehab | 350 | ✓ |
| 24 | 66007 | $28.90 | Plastic Surgery | 355 | ✓ |
| 25 | 70077 | $31.39 | General Surgery | 380 | ✓ |
| 26 | 70078 | $31.39 | General Surgery | 380 | ✓ |
| 27 | 78007 | $29.21 | Cardiac Surgery | 446 | ✓ |
| 28 | 78008 | $24.94 | Cardiac Surgery | 446 | ✓ |
| 29 | 79207 | $30.47 | Thoracic Surgery | 458 | ✓ |
| 30 | 79208 | $25.99 | Thoracic Surgery | 458 | ✓ |
| 31 | 08077 | $41.18 | Urology | 466 | ✓ |
| 32 | 08078 | $43.40 | Urology | 466 | ✓ |
| 33 | 94077 | $36.34 | Laboratory Medicine | 494 | ✓ |
| 35 | PG33241 | $15.51 | Endocrinology | 232 | ✓ |

---

## Required Fixes

### DELETE these rows:
- Row 0 (98010) — code doesn't exist
- Row 1 (98020) — code doesn't exist
- Row 34 (00040) — wrong code entirely

### UPDATE these rows:

**Row 4 (G14076):**
```
Fee: 15.00 → 21.92
```

**Row 5 (13037):**
```
Code: 13037 → P13037
Fee: 20.00 → 34.44
Description: → "Telehealth GP in-office Visit"
Page_Found: 142 → 114
```

**Row 6 (13018):**
```
Code: 13018 → P13018
Fee: 20.00 → 75.32
Description: → "Telehealth GP out-of-office Individual counselling (min 20 min)"
Page_Found: 142 → 115
```

**Row 17 (PG33250 video):**
```
modality: video → DELETE ROW (or change to "email")
```

**Rows 36-38 (Pediatric surcharges):**
```
Links_To: "50507, 50517" → "00510, 00550, 00551, 00585, 01511, 01512, 01513"
# OR delete these rows if they don't apply to telehealth
```

---

## Debug Checklist

```
[ ] Search PDF for "98010" — confirm no results
[ ] Search PDF for "98020" — confirm no results
[ ] Verify 00040 on page 59 — confirm stomach lavage
[ ] Verify P13037 on page 114 — confirm $34.44
[ ] Verify P13018 on page 115 — confirm $75.32
[ ] Verify G14076 on page 152 — confirm $21.92
[ ] Verify PG33250 on page 232 — confirm "telephone or email" only
[ ] Verify 50571-50573 linkages on page 335 — confirm in-person codes only
```
