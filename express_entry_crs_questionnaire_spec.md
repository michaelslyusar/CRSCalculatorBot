# Canada Express Entry CRS Calculator — Rules & Questionnaire Engineering Specification

**Rules verified:** 2026-10-16  
**Jurisdiction:** Canada / IRCC / Express Entry  
**Purpose:** Implement a transparent, versioned, explainable questionnaire and CRS scoring engine.  
**Status:** Engineering reference, not an official IRCC calculator or legal advice. **November 3, 2026 changes are announced but are not yet in force as of this document's verification date.**

> **Critical:** CRS (ranking score) is *not* the same as admission eligibility for one of the three Express Entry programs, the FSW 67/100 selection score, a provincial nomination eligibility assessment, or eligibility for a category-based draw. Implement these as separate outputs. Rules, occupation tables, language score conversions, eligibility policy and round criteria must be regularly synchronized with IRCC.

## Contents

1. Architecture and versioning
2. Questionnaire data contract
3. Relationships and scoring configuration
4. Age
5. Education
6. Language testing and scores
7. Canadian work experience
8. Spouse factors
9. Skills transferability
10. Additional CRS points
11. November 3, 2026 labour market integration
12. Calculation pipeline and pseudocode
13. Program eligibility (CEC, FSW, FST)
14. Category-based selections
15. Validation and tricky scenarios
16. Test cases and acceptance criteria
17. UI flow and explanations
18. Source registry and maintenance

---

## 1. Architecture and rule versions

### 1.1 Version selection

Implement a rules engine configured using the **date on which the score is assessed** (and, where applicable, the invitation date). Do **not** apply announced future policy as though currently effective.

| Rule version | Effective range | Canadian work experience | Job-offer CRS points | LMIA |
|---|---|---|---|---|
| `CRS_2025_03_25` | 2025-03-25 through 2026-11-02 | Main human-capital score (0–70 or 0–80) | 0 (abolished March 25, 2025) | Existing job-offer/program requirements remain separate |
| `CRS_2026_11_03` | From 2026-11-03 | Work-length moved into **Labour Market Integration**; potential high-wage bonus | 0/50/70/100 for genuine qualifying offers in listed high-wage NOCs; choose higher of experience route or offer route | No LMIA required for qualifying offers under new rules, subject to relevant transitional public policy and genuineness |

An updated IRCC online interface may lag legal effective dates. Preserve source URLs and rule date for every calculation. Scores can be recalculated when policy changes; an ITA uses applicable invitation-date rules, including the occupation-level snapshot for the new high-wage table.

### 1.2 CRS score structure

**Before 2026-11-03**:

```
core = age + education + first_language + second_language + canadian_experience
spouse = spouse_education + spouse_language + spouse_canadian_experience
transferability = min(100, education_transferability + foreign_transferability + trade_certificate_transferability)
additional = min(600, sibling + french_bonus + canadian_education + provincial_nomination)
TOTAL = core + spouse + transferability + additional  // overall cap 1200
```

**From 2026-11-03**:

```
core_without_work = age + education + first_language + second_language
spouse = spouse_education + spouse_language + spouse_canadian_experience
transferability = min(100, education_transferability + foreign_transferability + trade_certificate_transferability)
experience_route = canadian_work_length_points + qualifying_high_wage_canadian_experience_bonus
job_offer_route = qualifying_high_wage_job_offer_points
labour_market_integration = max(experience_route, job_offer_route)
additional = min(600, sibling + french_bonus + canadian_education + provincial_nomination)
TOTAL = min(1200, core_without_work + spouse + transferability + labour_market_integration + additional)
```

The new factor has maximum **140 with accompanying spouse** or **150 without accompanying spouse** on the experience route; job-offer maximum 100. The maximum total remains **1,200** under the existing overall CRS framework. Never add the experience and job-offer routes together. Canadian work experience still enters skills-transferability computations unchanged.

### 1.3 Mandatory outputs

Return structured fields, not just a number:

- `rules_version`, `assessment_date`, `source_checked_at`
- `crs_total` and exhaustive subtotals, including the winning labour-market route
- `program_eligibility`: CEC/FSW/FST as eligible, ineligible, or needs-review with reasons
- `pool_eligibility`: eligible only if at least one program can be confirmed and all global criteria are met
- `category_eligibility`: categories separately (eligible/ineligible/needs-review)
- `unverified_claims`, `validation_errors`, `review_flags`, `expiring_documents`
- `assumptions` and a per-line explanation (`points`, `rule_id`, `source`, `input_ids`)
- optionally `potential_score` for **hypothetical** future scenarios, prominently labelled not current CRS

**Do not label a person “Express Entry eligible” solely because the CRS score is high.**

---

## 2. Questionnaire data contract

Use a normalized schema with conditional questions. Recommended types:

```ts
type Skill = 'speaking' | 'listening' | 'reading' | 'writing';
type Language = 'en' | 'fr';
type CRSVersion = 'CRS_2025_03_25' | 'CRS_2026_11_03';
type EvidenceStatus = 'verified' | 'self_reported' | 'missing' | 'expired';
type Ternary = true | false | null;

type LanguageTest = {
  language: Language;
  test: 'CELPIP_GENERAL' | 'IELTS_GENERAL' | 'PTE_CORE' | 'TEF_CANADA' | 'TCF_CANADA';
  testDate: string; // YYYY-MM-DD
  scores: Record<Skill, number>; // raw test values, not presumed CLBs
  testReferenceNumber?: string;
  evidenceStatus: EvidenceStatus;
};

type Credential = {
  country: string;
  canadian: boolean;
  completionDate: string;
  type: 'secondary'|'one_year'|'two_year'|'three_plus'|'bachelors'|'masters'|'professional'|'doctorate'|'other';
  durationMonths?: number;
  institutionEligible?: Ternary;
  canadianStudyPhysicallyPresent?: Ternary;
  eligibleForCanadianEducationBonus?: Ternary;
  ecaStatus?: 'not_required'|'valid'|'expired'|'not_obtained'|'unknown';
  ecaCanadianEquivalency?: string;
  ecaIssueDate?: string;
  ecaProvider?: string;
  evidenceStatus: EvidenceStatus;
};

type WorkPeriod = {
  id: string;
  country: string;
  province?: string;
  physicallyInCanada: boolean;
  employerCanadian: boolean;
  startDate: string;
  endDate?: string;
  ongoing: boolean;
  noc2021: string; // FIVE characters, preserve leading zeros
  teer: 0|1|2|3|4|5;
  paid: boolean;
  legallyAuthorized: Ternary;
  selfEmployed: boolean;
  physicianPubliclyFundedException?: boolean;
  fullTimeStudentDuringWork: boolean;
  employmentType: 'employee'|'contractor'|'self_employed'|'unpaid';
  typicalHoursPerWeek: number;
  dutiesMatchNoc: Ternary;
  qualifiedToPracticeTrade?: Ternary;
  overlapsOtherJobs?: boolean;
  evidenceStatus: EvidenceStatus;
};

type Offer = {
  hasOffer: boolean;
  noc2021?: string;
  teer?: number;
  fullTime?: Ternary;
  continuous?: Ternary;
  nonSeasonal?: Ternary;
  durationAfterPRMonths?: number;
  canadianLocation?: Ternary;
  employerActive?: Ternary;
  genuine?: Ternary;
  employerCanFulfill?: Ternary;
  applicantQualified?: Ternary;
  writtenOffer?: Ternary;
  offerRevoked?: boolean;
  lmiaStatus?: 'positive'|'exempt'|'none'|'unknown';
  evidenceStatus: EvidenceStatus;
};

type Person = {
  dateOfBirth: string;
  citizenships: string[];
  permanentResidentOfCanada: boolean;
  credentials: Credential[];
  languageTests: LanguageTest[];
  work: WorkPeriod[];
  certificateOfQualificationCanadianTrade: boolean;
  validJobOffers: Offer[];
};

type Application = {
  assessmentDate: string;
  principalApplicant: Person;
  maritalStatus: 'single'|'married'|'common_law'|'separated'|'divorced'|'widowed';
  spouse?: Person;
  spouseAccompanying?: boolean;
  spouseCanadianCitizenOrPR?: boolean;
  provincialNomination?: { nominated: boolean; expressEntryAligned: boolean; accepted: boolean; valid: boolean; province?: string };
  sibling?: { exists: boolean; relationshipEligible: boolean; age: number; canadianCitizenOrPR: boolean; residesInCanada: boolean };
  intendsToResideOutsideQuebec: Ternary;
  inadmissibilityConcerns: Ternary;
  settlementFundsAvailableCAD?: number;
  proofOfFundsExempt?: Ternary;
};
```

**Keep raw user answers and calculated values separate.** Never allow users to manually overwrite an automatically derived CLB, counted years, high-wage NOC level, or eligibility classification without explicitly switching to an estimate-only mode.

---

## 3. Accompanying spouse configuration

Use `WITH_SPOUSE` when a spouse/common-law partner **is accompanying and is not already a Canadian citizen or PR**. Use `WITHOUT_SPOUSE` when applicant is unmarried, spouse is not accompanying, or spouse is Canadian citizen/PR. Do not simply use `maritalStatus !== 'single'`.

| Maximum baseline component | With accompanying spouse | Without accompanying spouse |
|---|---:|---:|
| Age | 100 | 110 |
| Education | 140 | 150 |
| First language | 128 | 136 |
| Second language | 22 | 24 |
| Canadian work length (old core / new integration) | 70 | 80 |
| Spouse factors | 40 | 0 |
| Skills transferability | 100 | 100 |
| Additional points | 600 | 600 |

Language combined cap is 150 with spouse / 160 without spouse, even when adding ability points for the second language.

Do not share or transfer the principal applicant's experience, job-offer bonus or nomination with the spouse's scoring entry. For couples offer a separate **swap principal applicant** calculation rather than incorrectly combining both adults' principal applicant entitlements.

---

## 4. Age points

Age = completed years on the scoring date, calculated using exact DOB and date (not just calendar year). Age is generally assessed when profile/ITA scoring is applied; application-stage age-lock rules warrant separate confirmation for immigration filing. Future birthdays may cause CRS score reduction.

| Age | With spouse | Without spouse |
|---|---:|---:|
| 17 or younger | 0 | 0 |
| 18 | 90 | 99 |
| 19 | 95 | 105 |
| 20–29 | 100 | 110 |
| 30 | 95 | 105 |
| 31 | 90 | 99 |
| 32 | 85 | 94 |
| 33 | 80 | 88 |
| 34 | 75 | 83 |
| 35 | 70 | 77 |
| 36 | 65 | 72 |
| 37 | 60 | 66 |
| 38 | 55 | 61 |
| 39 | 50 | 55 |
| 40 | 45 | 50 |
| 41 | 35 | 39 |
| 42 | 25 | 28 |
| 43 | 15 | 17 |
| 44 | 5 | 6 |
| 45+ | 0 | 0 |

**Age = 0 points ≠ automatically ineligible** for every program; run separate program eligibility.

---

## 5. Education points

### 5.1 Principal applicant core education

| Recognized highest category | With spouse | Without spouse |
|---|---:|---:|
| Less than high school | 0 | 0 |
| High school | 28 | 30 |
| One-year post-secondary credential | 84 | 90 |
| Two-year post-secondary credential | 91 | 98 |
| Bachelor's or 3+-year post-secondary | 112 | 120 |
| Two or more credentials; at least one 3+ years | 119 | 128 |
| Master's / eligible entry-to-practice professional degree | 126 | 135 |
| PhD | 140 | 150 |

Rules:
- A **Canadian** eligible credential normally needs no ECA. A **foreign** credential must have a valid immigration-purpose ECA to claim education points; expired or absent ECA = do not count that foreign credential.
- An ECA establishes the **Canadian equivalent**, which can differ from the diploma's foreign name. Use the precise ECA equivalency, not what user calls the diploma.
- An ECA normally has a **5-year** validity period. Verify issuance/expiration against assessment date.
- 'Two or more credentials' requires **at least two qualifying, completed and recognized credentials**, and at least one qualifying credential of 3+ years. A 2-year Canadian diploma plus a recognized 3+-year foreign degree is a possible example; a second unassessed foreign credential isn't automatically eligible.
- Professional degrees require the specific approved regulated entry-to-practice professions, not any professional-sounding credential.
- Education entered here must not be confused with the *separate* Canadian-education bonus below.

### 5.2 Spouse education

| Spouse recognized education | Points |
|---|---:|
| Less than high school | 0 |
| High school | 2 |
| One-year post-secondary | 6 |
| Two-year post-secondary | 7 |
| Bachelor's / 3+ year | 8 |
| Two or more; one 3+ year | 9 |
| Master's / qualifying professional degree | 10 |
| PhD | 10 |

Spouse education counts only in `WITH_SPOUSE` configuration, and foreign credentials similarly need appropriate ECA recognition.

---

## 6. Language: tests, CLB/NCLC conversion and CRS

### 6.1 Official languages and evidence

Accept only IRCC-recognized immigration tests: **CELPIP-General**, **IELTS General Training** (not Academic), **PTE Core**, **TEF Canada**, **TCF Canada**. Approved results must generally be **less than two years old** on both Express Entry profile submission and PR application dates. A booked future test or estimated CLB does **not** confer official CRS points. Keep hypothetical scores separate. French test skills are expressed in **NCLC**, English in **CLB**.

Convert **each of the four raw test abilities independently** using the appropriate test-specific official published lookup bands. Never average raw test scores or average CLBs. Some tests have different score thresholds for speaking, listening, reading and writing. Preserve the raw scores and mapping version.

**Do not implement CELPIP, IELTS, PTE, TEF, or TCF conversions using one approximate formula.** Link the official IRCC conversion tables in the Sources section; store each band's inclusivity, accepted score format, version and source date in a separate maintainable data table. In particular TEF score scales can differ by test reporting method and year.

### 6.2 Principal applicant's first official language

| CLB/NCLC **per ability** | With spouse | Without spouse |
|---|---:|---:|
| Under 4 | 0 | 0 |
| 4–5 | 6 | 6 |
| 6 | 8 | 9 |
| 7 | 16 | 17 |
| 8 | 22 | 23 |
| 9 | 29 | 31 |
| 10+ | 32 | 34 |

Sum speaking + listening + reading + writing. Max 128 with spouse / 136 without. Score English or French as first official language. The applicant's selection should be consistent with their tests and program thresholds; consider calculating and presenting each feasible first/second ordering if both languages are tested, rather than silently choosing one.

### 6.3 Second official language

| CLB/NCLC **per ability** | Points |
|---|---:|
| 4 or below | 0 |
| 5–6 | 1 |
| 7–8 | 3 |
| 9+ | 6 |

Sum four abilities; cap **22 with spouse**, **24 without spouse**. First+second combined language maximum **150 with spouse / 160 without**. If second language has no **valid** test, award zero.

### 6.4 Spouse's language

Use spouse's **first official language** (English or French), max 20 points total:

| CLB/NCLC **per ability** | Points |
|---|---:|
| 4 or below | 0 |
| 5–6 | 1 |
| 7–8 | 3 |
| 9+ | 5 |

Sum 4 abilities. Spouse's second official language **does not receive additional spouse CRS points**.

### 6.5 Language thresholds impact other point systems

- Transferability evaluates **all four** abilities of the principal applicant's **first** official language (details in Section 9).
- The separate **French bonus** requires **NCLC 7+ in every French ability**; English ability affects 25 vs 50 bonus (Section 10).
- Program minimums are distinct: FSW first-language CLB/NCLC 7 all four; CEC CLB/NCLC 7 for TEER 0/1 or 5 for TEER 2/3; FST speaking/listening 5 and reading/writing 4.
- French-category rounds may require NCLC 7 all four, but do not themselves add direct extra CRS points beyond French bonus and applicable language base points.

---

## 7. Counting Canadian and foreign work

### 7.1 Eligibility for CRS-counted Canadian work

- Experience in a qualifying **NOC 2021 TEER 0, 1, 2 or 3** occupation; select NOC according to **duties**, not job title alone.
- Paid employment, legally authorized and **physically performed in Canada** for a Canadian employer; Canadian employer + work physically outside Canada is **not** Canadian work for CRS.
- Work in prior **10 years** for CRS, with **30 hours per week maximum across concurrent jobs**. 1 full-time-equivalent year requires at least 12 months and 1,560 counted hours; additional weekly hours cannot accelerate the minimum one calendar year.
- Canadian work while a **full-time student** (including co-op) does **not** count as CRS Canadian experience; foreign student work can count for applicable foreign experience.
- Canadian self-employment generally **does not** count; check current special public policy for certain physicians providing publicly funded medical services.
- Work need **not be continuous** to receive Canadian CRS work-length points; distinguish from FSW continuity requirements.
- Part-time and separate jobs may aggregate; cap shared weeks at 30 hrs total, account for overlapping dates and time-zone/boundary issues.
- For CEC minimum eligibility, count qualifying employment within **3 years** rather than CRS's 10-year window.

### 7.2 Canadian work length points

These exact points are a core component under the **pre-November-3** CRS, and a subcomponent of **Labour Market Integration** from November 3 onward.

| Completed Canadian FTE years | With spouse | Without spouse |
|---|---:|---:|
| Less than 1 | 0 | 0 |
| 1 | 35 | 40 |
| 2 | 46 | 53 |
| 3 | 56 | 64 |
| 4 | 63 | 72 |
| 5+ | 70 | 80 |

**Caution:** Do not reduce multi-job work to `sum(hours)/1560` alone; full-time-equivalent **elapsed periods** must also satisfy the 12-month-per-year constraint and the 30-hours/week cap.

### 7.3 Foreign work experience

No standalone CRS core points are given simply for foreign work. Instead, eligible foreign skilled work feeds **skills transferability**. Work must be properly NOC classified and independently verified against the relevant 10-year/eligibility definitions. Calculate aggregate foreign skilled experience as 0, 1–2, or 3+ qualifying years for transferability. Full-time student work **outside Canada** can qualify under applicable policy; paid self-employment outside Canada may qualify with sufficient evidence, unlike ordinary Canadian self-employment.

Foreign paid work performed remotely **while physically in Canada for a foreign employer** must not be arbitrarily classified as Canadian experience. Review its proper classification under applicable IRCC definitions; flag for manual review rather than automatically treating it as Canadian employer work.

---

## 8. Spouse Canadian experience

Only if `WITH_SPOUSE`:

| Canadian years | Points |
|---|---:|
| Under 1 | 0 |
| 1 | 5 |
| 2 | 7 |
| 3 | 8 |
| 4 | 9 |
| 5+ | 10 |

Spouse subtotal = spouse education (max 10) + spouse first-language (max 20) + spouse Canadian work (max 10), at most **40**. Spouse foreign employment does **not** earn principal applicant's skill transferability points; high-wage work of spouse does **not** grant principal applicant's high-wage bonus.

---

## 9. Skills-transferability combinations — maximum 100 total

**Key rule:** These are *combinations*, not a sum of the headline maximums. Education group max 50, foreign-work group max 50, certificate group max 50, and **overall transferability max 100**. Compute subcategories as specified and cap groups.

### 9.1 Education × first language

| Recognized education | First language all four CLB/NCLC 7+, at least one under 9 | All four 9+ |
|---|---:|---:|
| High school or less | 0 | 0 |
| One qualifying post-secondary credential (1+ year), including bachelor's | 13 | 25 |
| Two+ credentials with one 3+ years / qualifying master's / qualifying professional / PhD | 25 | 50 |

If **any** of the four first-language abilities is below 7, this combination yields **0**.

### 9.2 Education × Canadian work

| Education | 0 Canadian years | 1 Canadian year | 2+ Canadian years |
|---|---:|---:|---:|
| High school or less | 0 | 0 | 0 |
| One qualifying post-secondary credential (1+ year) | 0 | 13 | 25 |
| Two+ (one 3+ years) / master's / professional / PhD | 0 | 25 | 50 |

**Education transferability subtotal = `min(50, education_language + education_canadian_experience)`**.

### 9.3 Foreign work × first language

| Eligible foreign work | First language below 7 in any ability | All four 7+, one under 9 | All four 9+ |
|---|---:|---:|---:|
| 0 | 0 | 0 | 0 |
| 1–2 years | 0 | 13 | 25 |
| 3+ years | 0 | 25 | 50 |

### 9.4 Foreign work × Canadian work

| Foreign work | 0 Canadian years | 1 Canadian year | 2+ Canadian years |
|---|---:|---:|---:|
| 0 | 0 | 0 | 0 |
| 1–2 years | 0 | 13 | 25 |
| 3+ years | 0 | 25 | 50 |

**Foreign work transferability subtotal = `min(50, foreign_language + foreign_canadian_experience)`**.

### 9.5 Canadian trade certificate × first official language

Requires an eligible **certificate of qualification in a trade occupation** issued by a Canadian federal, provincial or territorial authority.

| First language abilities | Points |
|---|---:|
| Any of four below CLB/NCLC 5 | 0 |
| All four 5+, but at least one below 7 | 25 |
| All four 7+ | 50 |

**Total transferability = `min(100, education_subtotal + foreign_subtotal + trade_subtotal)`.**

Example: Education-language 50 + education-Canadian 50 = education subtotal **50**, not 100. Foreign-language 50 + foreign-Canadian 50 = foreign subtotal **50**, not 100. Combined total 100.

---

## 10. Additional CRS points — overall cap 600

### 10.1 Provincial nomination

**+600** only for a **valid, accepted Express Entry-aligned provincial/territorial nomination**. Non-Express Entry provincial streams do not automatically provide Express Entry CRS nomination points. Do not treat a pending expression of interest or application as an issued/accepted nomination. Verify validity and intended province of residence. With +600, the aggregate **additional points category** remains capped at 600, even with other bonuses.

### 10.2 Qualifying sibling in Canada

**+15** if candidate or accompanying spouse/common-law partner has a qualifying brother/sister (including eligible step/adoptive relationships per official requirements) who is **18+**, **lives in Canada**, and is a **Canadian citizen or permanent resident**. Multiple siblings still yield a maximum **15**.

### 10.3 French proficiency bonus

| French | English | Points |
|---|---|---:|
| All four French skills at NCLC 7+ | No English test, or CLB 4 or below across English skills | 25 |
| All four French skills at NCLC 7+ | All four English skills CLB 5+ | 50 |
| Any French skill under NCLC 7 | Any | 0 |

**Edge case:** IRCC's main score table lists the 25-point case as English CLB 4 or lower (or no test) and the 50-point case as English CLB 5+ in all four. If English scores mix <5 and >=5, **verify the governing regulatory text** before awarding a bonus (do not arbitrarily interpolate or award 50). Keep flagged `needs_review` until legally reconciled.

The French bonus can apply regardless of whether French is nominated first or second official language, subject to valid tests.

### 10.4 Canadian post-secondary education bonus

| Eligible completed Canadian post-secondary credential | Points |
|---|---:|
| No qualifying credential | 0 |
| One-/two-year qualifying Canadian credential | 15 |
| Eligible credential from program of three years or longer | 30 |

**Not cumulative**: max 30, not 15 + 30. Credential eligibility requires meeting the published conditions, which generally include study at an eligible Canadian institution and physical presence in Canada for the required period. **Some distance learning, English/French-as-a-second-language study and scholarship obligations may make the credential ineligible for this bonus.** Do not assume every Canadian credential qualifies; implement detailed education-eligibility questionnaire using the current official page and regulation.

### 10.5 No regular CRS points for jobs (prior version)

Since **2025-03-25**, job offers receive **0 direct CRS points** under the pre-November-3 system. Historical 50/200 job-offer rules must not be applied during this date interval. Job offers can still affect **FSW/FST program eligibility** and other programs.

---

## 11. From November 3, 2026: Labour Market Integration (LMI)

**Announced 2026-10-16; effective 2026-11-03.** This replaces the location of the existing Canadian work-length points and adds new points for high-wage experience or a qualifying high-wage offer.

### 11.1 High-wage NOC table (external and versioned)

Use the **exact IRCC high-wage occupation NOC list**. Categories are derived from occupation-wide national median wages, **not the particular user's salary**:

| Tier | Occupation threshold against overall Canadian median wage | Qualifying Canadian experience bonus | Qualifying job offer points |
|---|---|---:|---:|
| `TIER_2_0` | At least 2.0× | 70 | 100 |
| `TIER_1_5` | At least 1.5× and below 2.0× | 40 | 70 |
| `TIER_1_3` | At least 1.3× and below 1.5× | 20 | 50 |
| `NOT_LISTED` | Not listed | 0 | 0 |

- Match five-character **NOC 2021** codes against IRCC's published list effective on assessment/invitation date.
- The official list initially uses wage evidence from **2023–2025**, taking the highest qualifying level in the past three years. Store `effectiveFrom`, `effectiveTo`, `sourceDate`, `tier`, and `nocCode`.
- Review updated tiers when released. **Once ITA issued, freeze the tier applicable on ITA date**, even if list changes before application decision (per IRCC).
- Never infer tier from personal hourly pay, job title string, or the previous year's NOC lists.
- An occupation need not be in the STEM category-based invitation list to appear on the high-wage points list; these are separate lists.

### 11.2 High-wage Canadian experience bonus

At least **12 months / 1,560 counted hours** of qualifying Canadian skilled work in one or more listed high-wage NOCs; periods can be non-continuous and across different jobs, applying the same 30-hours/week concurrent cap and other CRS Canadian-work exclusions.

Algorithm: 

1. For highest tier, aggregate qualifying Canadian work in `TIER_2_0` only; if 12 months/1,560 eligible hours, bonus **70**.
2. Otherwise aggregate qualifying work in `TIER_2_0` + `TIER_1_5`; if threshold met, bonus **40**.
3. Otherwise aggregate qualifying work in **all three tiers**; if threshold met, bonus **20**.
4. Otherwise bonus **0**.

Important: do not simply assign bonus based on the tier of the applicant's current or longest-running job. Cross-job combinations are allowed. Do not sum tier bonuses. A person with 11 months in tier 2.0 and 1 month tier 1.5 may qualify for **40**, not 70, assuming independently compliant hours and elapsed time.

`experience_route = regular_canadian_work_length_points + high_wage_bonus`.

### 11.3 Job offer route

A qualifying job offer must generally be **full-time, continuous work in Canada for at least one year after PR**, and must meet applicable profession/program requirements. Officers determine **genuineness**; consider employer actually conducting business, its genuine staffing needs, ability to fulfill the terms, employment-law compliance, the candidate's qualifications, and likelihood the candidate will accept and perform the employment. Revoked offers do not qualify. Some rules are different for eligible trades/physicians.

From 2026-11-03, an **LMIA is not required** for qualifying offers under the published new rules and public policy. This does **not** mean merely uploading an informal email or invented contract guarantees an offer. It also does **not** make a TEER 4/5 offer eligible for FSW/FST minimum requirements. Don't treat a job offer with the wrong NOC, part-time arrangement or inadequate future duration as awardable.

`offer_route = max(100|70|50|0 across validated offers according to tier)`.

- For **FSW/FST eligibility**, genuine offers in eligible TEER 0–3 occupations may matter even if NOC is **not** high-wage and receives **0 LMI offer points**.
- Whether an offer is eligible for the **program** and whether it earns **CRS high-wage points** are different checks.
- Temporary IRCC Express Entry-profile **LMIA workaround** for no-LMIA offers after 2026-11-03 belongs in a help article, **not** calculator logic: IRCC says answer LMIA = Yes, number `00000`, issue `2026-01-01`, expiry `2029-01-01`, while their interface is being updated. This is a policy-specific UI workaround, not a real LMIA, and should be removed once IRCC updates the system.

### 11.4 Winner selection

```
LMI = max(experience_route, offer_route)
LMI_winner = experience_route >= offer_route ? 'experience' : 'job_offer'
```

Do not sum the two routes. At a tie, show both routes are equivalent but still credit LMI **once**. Separately retain regular Canadian experience duration for skills-transferability computation.

### 11.5 Worked LMI examples

| Scenario | With spouse | Without spouse |
|---|---:|---:|
| One year Canadian TEER 0–3, NOC not high-wage, no qualifying offer | 35 | 40 |
| One year entirely tier 1.3 Canadian work | 35+20 = **55** | 40+20 = **60** |
| One year entirely tier 1.5 Canadian work | 35+40 = **75** | 40+40 = **80** |
| One year entirely tier 2.0 Canadian work | 35+70 = **105** | 40+70 = **110** |
| Five years Canadian TEER 0–3 + qualifying year tier 2.0 | 70+70 = **140** | 80+70 = **150** |
| No Canadian work; qualifying offer tier 1.5 | **70** | **70** |
| One year Canadian tier 1.5 + offer tier 2.0 | max(75,100) = **100** | max(80,100) = **100** |
| Five years Canadian non-high-wage + tier 1.3 offer | max(70,50) = **70** | max(80,50) = **80** |

---

## 12. Calculation order and implementation pseudocode

### 12.1 Recommended order

1. Load versioned policy snapshots for the date, including language conversion tables and high-wage NOC tiers.
2. Validate application data, DOB, tests and ECA expiries and that a valid principal applicant profile exists.
3. Normalize work periods and mark reason-coded inclusions/exclusions for each scoring or program window.
4. Resolve spouse configuration (accompanying vs non-accompanying or Canadian citizen/PR).
5. For each valid principal applicant first-language choice, convert results per ability and score base language/transferability/French bonus, retaining the supported best permissible configuration.
6. Determine recognized education equivalencies, core/spouse education and Canadian bonus separately.
7. Compute age.
8. Compute Canadian and foreign qualifying years. Independently compute CEC 3-year, FSW 10-year continuous, and FST 5-year periods.
9. Compute spouse points if applicable.
10. Compute transferability group caps and global 100 cap.
11. Compute additional points with nomination cap 600.
12. Calculate Canadian work according to **rules version**. After 2026-11-03 calculate LMI higher-of-two-routes.
13. Produce score breakdown and reason codes. Independently calculate program eligibility and category eligibility.
14. Generate future simulation scenarios separately. Compare scores to latest applicable *published draws* only when those data are fetched; **never guarantee an invitation**.

### 12.2 TypeScript pseudocode

```ts
function calculateCRS(input: Application, policy: RulesSnapshot): CRSResult {
  const withSpouse = determineSpouseMode(input);
  const age = agePoints(input.principalApplicant.dateOfBirth, input.assessmentDate, withSpouse);
  const credentials = eligibleCredentials(input.principalApplicant.credentials, input.assessmentDate);
  const education = educationPoints(credentials, withSpouse);
  const languages = evaluateApprovedLanguageTests(input.principalApplicant.languageTests, policy);
  const selected = selectPermissibleFirstLanguage(languages, input, policy);
  const firstLanguage = firstLanguagePoints(selected, withSpouse);
  const secondLanguage = secondLanguagePoints(selected, withSpouse);

  const canadianWork = aggregateQualifyingCanadianWork(input.principalApplicant.work, input.assessmentDate);
  const foreignWork = aggregateQualifyingForeignWork(input.principalApplicant.work, input.assessmentDate);
  const canadianYears = canadianWork.completedFteYears;
  const workLength = canadianWorkLengthPoints(canadianYears, withSpouse);

  const spouse = withSpouse ? scoreSpouse(input.spouse!, input.assessmentDate) : 0;
  const transferability = scoreSkillTransferability({
    selected, credentials, canadianYears, foreignWork,
    certificate: input.principalApplicant.certificateOfQualificationCanadianTrade
  });
  const additional = Math.min(600, scoreSibling(input) + scoreFrenchBonus(selected)
    + scoreCanadianEducation(credentials) + scoreNomination(input));

  let labourMarketIntegration = 0;
  let coreWork = workLength;
  if (policy.effectiveFrom >= '2026-11-03') {
    coreWork = 0;
    const bonus = scoreHighWageCanadianExperience(canadianWork, policy.highWageNocs);
    const experienceRoute = workLength + bonus;
    const offerRoute = scoreHighestValidOffer(input.principalApplicant.validJobOffers, policy);
    labourMarketIntegration = Math.max(experienceRoute, offerRoute);
  }
  const rawTotal = age + education + firstLanguage + secondLanguage + coreWork
    + spouse + transferability + additional + labourMarketIntegration;
  return {
    crsTotal: Math.min(1200, rawTotal),
    breakdown: { age, education, firstLanguage, secondLanguage, coreWork,
      spouse, transferability, additional, labourMarketIntegration },
    // separately: calculateCEC(), calculateFSW(), calculateFST(), categoryEligibility()
  };
}
```

Pseudocode intentionally omits detailed legal validations. **Do not ship it unchanged as production scoring code.** Rules version comparison should use explicit enum/date, not merely a naive lexical `effectiveFrom` comparison if overlapping snapshots or future versions become possible.

---

## 13. Program eligibility — separate from CRS

**Express Entry pool membership requires minimum eligibility under at least one of CEC, FSW or FST**, and applicable admissibility, intentions and other conditions. The tables below guide pre-screening; mark disputed evidence as `needs_review` rather than incorrectly rejecting or guaranteeing eligibility.

### 13.1 Common checks

- Candidate intends to live **outside Quebec** under federal Express Entry permanent-resident selection. Experience acquired **in Quebec** can still count where otherwise eligible. Require evidence of future non-Quebec settlement if currently in Quebec.
- Candidate meets admissibility requirements; self-reported answer cannot fully determine medical/criminal/security admissibility.
- NOC duties and TEER match, including lead statement and most main duties, not just code/title.
- Language tests and educational assessments still valid when required; document validity at relevant date.
- Where proof of funds is necessary, verify against the **current IRCC family-size table** and exemptions (do not hardcode amounts that change yearly). Certain eligible CEC applicants and workers authorized in Canada with qualifying offer may be exempt under applicable regulations; check exact conditions.

### 13.2 Canadian Experience Class (CEC)

- At least **1 year / 1,560 eligible hours within the past 3 years**, TEER **0–3**, paid, physically worked in Canada with authorization under temporary status, matching duties. Work can span multiple qualifying jobs/NOCs; not necessarily continuous.
- **TEER 0/1**: minimum **CLB/NCLC 7 each** in all four abilities; **TEER 2/3**: minimum **5 each**. If qualifying work spans TEER categories, validate the rule applied to the occupations the candidate relies on; flag mixed-band ambiguities.
- **No formal education requirement.**
- Full-time-student work (including co-op), unauthorized/unpaid work, and ordinary Canadian self-employment do not count (specific physician exception possible).
- Plan to settle outside Quebec and be admissible.

### 13.3 Federal Skilled Worker Program (FSW)

- At least **one continuous year** (1,560 eligible hours and at least 12 months) in a **single NOC** TEER **0–3** within the past **10 years**. The experience must be paid and meet required duties. Eligible paid student employment can count if continuous, and eligible foreign self-employment may count. Part-time equivalence subject to continuity.
- First official language **CLB/NCLC 7 all four**, accepted test.
- Canadian secondary/post-secondary credential or foreign credential with valid **immigration-purpose ECA**.
- Score **at least 67/100** on FSW *selection-factor* grid below, **distinct from CRS**.
- Meet settlement funds requirement unless exempt, intend to settle outside Quebec, admissible.

#### FSW 67-point selection grid

| Factor | Maximum | Details |
|---|---:|---|
| First language | 24 | CLB 7 = 4 per ability, CLB 8 = 5, CLB 9+ = 6; below 7 in any ability ineligible |
| Second language | 4 | +4 if at least CLB/NCLC 5 in **all four** abilities; else 0 |
| Education | 25 | See separate education table below |
| Skilled work | 15 | 1 year 9; 2–3 years 11; 4–5 years 13; 6+ years 15 |
| Age | 12 | 18–35: 12; each year 36–46 subtract 1; 47+: 0; under 18: 0 |
| Arranged employment | 10 | Up to 10 for qualifying offer under FSW requirements; **distinct from CRS job-offer points** |
| Adaptability | 10 | Sum eligible indicators, cap 10; conditions below |
| **Total** | **100** | Minimum **67** |

FSW education grid:

| Education | FSW points |
|---|---:|
| PhD | 25 |
| Master's or eligible professional degree | 23 |
| Two+ post-secondary credentials, one 3+ years | 22 |
| Bachelor's / 3+ years | 21 |
| Two-year post-secondary | 19 |
| One-year post-secondary | 15 |
| High school | 5 |
| Below high school | 0 |

FSW adaptability (max **10**):

| Item | Points | Conditions / checks |
|---|---:|---|
| Spouse/partner language | 5 | Spouse CLB/NCLC 4+ in all four approved-test abilities |
| Candidate previous Canadian study | 5 | Minimum two academic years of eligible full-time Canadian study; verify policy conditions |
| Spouse previous Canadian study | 5 | Same basic two-academic-year qualifying criterion |
| Candidate previous Canadian work | 10 | At least one qualifying year of full-time skilled Canadian work |
| Spouse previous Canadian work | 5 | At least one qualifying year of full-time Canadian work meeting FSW adaptability conditions |
| Arranged employment | 5 | Meets FSW arranged-employment criteria |
| Qualifying relative in Canada | 5 | Eligible relationship, adult Canadian citizen/PR resident in Canada; relationships broader than CRS sibling bonus |

**FSW caution:** The October 16 announcement changes the LMIA requirement for qualifying offers effective November 3 under a temporary public policy, but the arranged-employment *selection grid* has its own rules. Keep arranged-employment eligibility versioned and source-verified; do not blindly use the CRS high-wage tier logic or remove the FSW 10-point factor.

### 13.4 Federal Skilled Trades Program (FST)

- **At least 2 years / 3,120 counted hours** paid trade experience within **5 years** before application, in qualifying skilled-trade NOC groups (2021 NOC: major groups 72 except sub-major 726; 73; 82; 83; 92; 93 except sub-major 932; minor group 6320; unit group 62200). Work must satisfy the NOC requirements and be in an occupation the applicant was qualified to practise.
- Minimum language: **speaking and listening CLB/NCLC 5**, **reading and writing CLB/NCLC 4**.
- Either qualifying full-time **job offer for at least one year** or **Canadian certificate of qualification** in skilled trade.
- No formal education minimum. Work gained as a full-time student does not count toward FST eligibility.
- Outside-Quebec settlement intent and admissibility requirements. Check any additional applicable rules or public policies.

### 13.5 Decision model

```
CEC: eligible | ineligible | needs_review
FSW: eligible | ineligible | needs_review
FST: eligible | ineligible | needs_review
poolEligibility: eligible if at least one is eligible AND global checks pass
```

If no program is verified eligible but some have missing evidence, show **"Eligibility not yet established"**, not automatically ineligible. If user currently fails all confirmed requirements, show potential CRS for education/planning, distinctly marked **hypothetical**.

---

## 14. Category-based invitation rounds ≠ CRS points

Categories can affect *which invitation round* someone is considered for, **not add a static number to CRS**. IRCC may change categories and NOC membership independently from CRS high-wage tiers. Track:

- `categoryId`, `name`, `validFrom`, `validTo`, `eligibleNocs`, `minimumWorkMonths`, `lookbackMonths`, `canadianOnly`, `languageRequirements`, `roundSpecificInstructionsUrl`.
- For 2026, published categories include French proficiency; healthcare/social services; education; STEM; trades; and new categories such as physicians with Canadian experience, researchers with Canadian experience, senior managers with Canadian experience, transport occupations and some Canadian Armed Forces-related recruitment.
- For **French-language proficiency category**, require **NCLC 7 in all four** and general Express Entry pool eligibility.
- For many occupation categories, IRCC requires **12 months full-time or equivalent** in **one eligible occupation** within **past 3 years**; the time may be noncontinuous, and Canada-vs-abroad depends on category. **Do not combine time across different NOCs** where the category says a single occupation.
- Being a software developer **does not automatically mean** eligible for the STEM category: use the actual dated STEM NOC list. For example, cybersecurity specialist NOC `21220` appears on the published STEM list; software developer NOC `21232` is not on that particular published list as of verification.
- Every category round can have extra criteria and NOC amendments. Never calculate a 'category CRS bonus'.

---

## 15. Validation rules and edge cases

### 15.1 User input validation

| ID | Input | Validation / behavior |
|---|---|---|
| `VAL_DATE` | Dates | ISO date, not future for completed event, DOB before assessment, no end before start |
| `VAL_SPOUSE` | Spouse | Require accompaniment and Canadian status; only apply spouse points if correct scoring configuration |
| `VAL_TEST` | Test | Only approved test + four scores + valid date + official per-test conversion |
| `VAL_ECA` | Foreign education | Valid ECA where needed; ECA result controls equivalency |
| `VAL_WORK` | Work | 5-character NOC, TEER, dates, location, pay, status, student work, weekly hours, employer and duties |
| `VAL_OVERLAP` | Concurrent work | Max 30 hours/week counted across all jobs; no double counting |
| `VAL_OFFER` | Job offer | Separate employment offer from eligibility/points; genuineness and duration required |
| `VAL_NOMINATION` | PNP | Express Entry-aligned, accepted and valid, not simply issued EOI |
| `VAL_HIGH_WAGE` | NOC tier | Query effective-date IRCC NOC snapshot; no salary-derived tier |
| `VAL_FRENCH` | Mixed English CLB | Verify 25-point boundary treatment; don't grant unsupported 50 points |
| `VAL_ASSESSMENT` | Dates of policy | Do not apply November 2026 rules in October 2026 unless explicitly forecasting |
| `VAL_QUEBEC` | Settlement | Quebec work can count but federal settlement intent must be outside Quebec |

### 15.2 Edge cases to explicitly support

1. Married applicant, spouse **not accompanying** → score without accompanying spouse; no spouse points.
2. Married applicant, spouse is Canadian PR/citizen → score without accompanying spouse.
3. Foreign degrees + Canadian diploma → recognized combined 'two or more' only if valid ECA and 3+-year credential conditions satisfied.
4. Two years of Canadian work while studying full-time → zero CRS Canadian work, despite pay.
5. Paid work from home in Canada for a US-only employer → don't automatically count as Canadian experience.
6. Concurrent 25+25 hours/week → count at most 30 total per calendar week.
7. 1,560 hours in nine months at 40+/week → does not meet 12-month minimum.
8. Five years TEER 4 work → no skilled CRS Canadian work-length points.
9. First language CLB [9,9,9,8] → not 'all CLB 9'; use lower transferability bracket.
10. French [7,7,7,6] → French bonus 0 and no French-category qualification.
11. Valid offer in non-high-wage TEER 2 after Nov 3 → 0 LMI *offer* points, may still support FSW/FST eligibility.
12. Current qualifying high-wage tier 1.5 work 11 months → no high-wage bonus, though another qualifying non-high-wage period may contribute to overall Canadian work length.
13. Tier 2.0 work 11 months + tier 1.5 work 1 month, no overlaps → can give tier 1.5 bonus 40 after 12 eligible months; not 70.
14. One year Canadian tier 1.5 plus tier 2.0 offer → max(75,100) with spouse, **not 175**.
15. 600 PNP + 50 French + 30 education + 15 sibling → additional group **600**, not 695.
16. Two Canadian diplomas each two years → does not meet two-or-more requiring one credential of three years, unless ECA/other recognized credential qualifies.
17. CEC qualifies after a year of Canadian work, but FSW's one-year **continuous** same-NOC rule may fail.
18. Valid test today expires before expected ITA → display risk; do not pretend future validity.
19. Multiple eligible language exams → apply valid result rules, avoid cherry-picking individual ability scores from different tests unless explicitly permitted by IRCC.
20. Category NOC list changed after candidate's profile creation → recalculate category eligibility for specific round date; don't assume category is fixed.
21. High-wage NOC list updated after ITA → hold invitation-date tier for that application.
22. Offer rescinded → job-offer route disappears; eligibility may also change.
23. Offer from startup: registered company + contract alone not conclusive; genuineness and ability to pay/operate are relevant.
24. A spouse's high-wage offer does not provide principal applicant LMI job-offer route points.
25. Overseas remote work for Canadian firm → not automatically Canadian work experience.
26. Work in Quebec with intention to settle Ontario → may count toward CEC subject to evidence of intended settlement outside Quebec.
27. Same individual alternate role as PA vs spouse → recompute everything including age, work, languages, ECA, nomination, funds, and eligibility; cannot simply swap totals.
28. CLB/NCLC 10+ → same CRS language points as 10, even if official test scale distinguishes higher.
29. Job offer 'LMIA number 00000' workaround → never represent it as a real LMIA or as evidence of approved offer.
30. Assessment after rule change: default to in-force version; display as-of date in results.

---

## 16. Automated test fixtures and acceptance criteria

### 16.1 Unit-test boundaries

**Age**: 17→18, 19→20, 29→30, 44→45, birthday tomorrow.

**Language**: each test's threshold just below / exactly at / just above each CLB band; all-four constraints; language 1 vs 2 ordering; 22-point second-language cap with spouse; expiration exactly at two-year boundary.

**Education**: every row; ECA expired; duplicate credentials; master's vs professional; two-or-more conditions.

**Work**: exactly 1,559 vs 1,560 hours, exactly 11 vs 12 months, 30 vs 40 h/week, concurrent work, student exclusion, self-employment, CEC three-year and CRS ten-year boundaries.

**Transferability**: no one at/above CLB 7 vs four all 7; all 9; education-group cap 50; foreign-group cap 50; global cap 100.

**Additional**: French NCLC threshold, sibling age 17 vs 18, Canadian study 1/2/3-year cases, PNP cap 600.

**LMI**: 0/20/40/70 experience tiers; 0/50/70/100 offer tiers; tier-mixed aggregation; max branch; spouse/no-spouse cap; date switches 2026-11-02 vs 2026-11-03.

**Eligibility**: CEC vs FSW vs FST examples; FSW 66 vs 67; CEC mixed NOCs; FST trade NOC valid/invalid; Quebec intention; missing authorization.

### 16.2 Golden scoring fixtures

All examples are **component tests**, not complete CRS totals unless every input is specified.

| Test | Expected |
|---|---|
| Age 30, with spouse | 95 age pts |
| Age 30, without spouse | 105 age pts |
| Principal two-year diploma, with spouse | 91 education pts |
| Principal bachelor's, no spouse | 120 education pts |
| Principal first language all CLB 10, with spouse | 128 first-language pts |
| Principal first language all CLB 9, no spouse | 124 first-language pts |
| Spouse first language all CLB 9 | 20 spouse language pts |
| French all NCLC 7 + English all CLB 5 | 50 French additional pts |
| 1 year Canadian, with spouse, pre-Nov | 35 Canadian core pts |
| 1 year Canadian in tier 1.5, with spouse, Nov 3+ | 75 LMI pts |
| 2 years Canadian non-high-wage, without spouse, Nov 3+ | 53 LMI pts |
| 5 years Canadian tier 2.0, without spouse, Nov 3+ | 150 LMI pts |
| Tier 1.5 offer, no work, Nov 3+ | 70 LMI pts |
| Tier 2.0 offer + 1 year tier 1.5 work, with spouse, Nov 3+ | 100 LMI pts |
| PNP + sibling + French bonus | 600 total additional pts |
| Education-language 50 and education-work 50 | 50 education transferability pts |

### 16.3 Acceptance criteria

- Every displayed point links to at least one underlying question and rule ID.
- Exact policy effective date is printed prominently.
- Current and hypothetical future scores are visually and semantically distinct.
- Invalid/unconfirmed evidence never silently becomes confirmed points.
- Test results are deterministic and reproducible against a rules snapshot ID.
- Version transitions don't double-count Canadian experience.
- All NOC and language conversion data can be updated without deploying a new scoring algorithm.
- Test against official IRCC calculator sample profiles where feasible, and manually reconcile differences before release.

---

## 17. Recommended questionnaire UX

### Steps

1. **Purpose & calculation date:** Current score or future estimate; show active version.
2. **Principal applicant & family:** DOB, marital/common-law, spouse status, accompanying or not, Canadian PR/citizen spouse, intent to reside outside Quebec.
3. **Language tests:** Approved test, date, four raw ability scores; guide with official test score labels; show CLB/NCLC live.
4. **Education:** Add each Canadian/foreign credential, ECA details and Canadian-study bonus validation questions.
5. **Employment history:** Repeatable employment periods, country/physical location, NOC duty finder, dates, authorized/paid/student/self-employment, weekly hours, overlaps.
6. **Offer:** Written offer, employer, NOC, duration after PR, full-time/continuous, genuineness indicators. Do not demand LMIA after Nov 3.
7. **Spouse details:** Ask only where relevant, mirror education/language/employment capture.
8. **Additional:** Valid EE-aligned PNP nomination, eligible siblings, Canadian certificates of qualification.
9. **Program eligibility add-on:** Source of funds / exemption, FSW arranged-employment/adaptability, trade certificates, intent and admissibility caveats.
10. **Results:** Total score, waterfall breakdown, program eligibility, scenario planner and evidence gaps.

### Output copy examples

- **"Your current estimated CRS: 462 (rules effective Oct 16, 2026)."** Only show example numeric value if it is actually calculated from real submitted data.
- **"Nov 3 scenario: +40 high-wage experience points, if your Canadian NOC and one-year employment satisfy the official requirements."**
- **"This offer contributes 0 CRS offer points because its NOC is absent from the current high-wage list; it may nevertheless count toward FSW eligibility."**
- **"You have 1,560 reported hours but under 12 elapsed months; no full year counted yet."**
- **"We cannot establish CEC eligibility until we know whether you were a full-time student during this employment."**

### Accessibility/privacy

- Progressive disclosure, plain-language NOC tooltips, mobile-first forms, accessible error summaries and keyboard operation.
- Don't require unnecessary medical, criminal, passport or personally identifying detail for initial estimate.
- Keep precise date of birth and work-history records private; offer anonymous/offline estimate mode where possible; avoid storing test IDs except for authorized workflow.
- Explain that CRS points and possible invitation outcomes are not guarantees.

---

## 18. Sources and ongoing maintenance

**Official primary sources (checked 2026-10-16):**

1. [IRCC — Comprehensive Ranking System (CRS) criteria](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/check-score/crs-criteria.html) — base, spouse, transferability, additional tables.
2. [IRCC — November 3, 2026 changes](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/changes.html) — Labour Market Integration, high-wage bonuses, job offers, LMIA changes and temporary workaround.
3. [IRCC — High-wage occupations and levels](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/check-score/high-wage.html) — authoritative versioned NOC list. **Import the complete list from this page; do not hardcode a partial IT list.**
4. [IRCC — High-wage occupations policy backgrounder, Oct 16, 2026](https://www.canada.ca/en/immigration-refugees-citizenship/news/2026/10/changes-to-express-entry-for-high-wage-occupations.html).
5. [IRCC — Job offers](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/documents/job-offer.html).
6. [IRCC — Language test results and equivalencies](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/documents/language-test.html) — complete band-to-CLB/NCLC conversions for supported tests.
7. [IRCC — Educational credential assessment](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/documents/education-assessment.html) — ECA and recognized equivalency.
8. [IRCC — Canadian Experience Class](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/who-can-apply/canadian-experience-class.html).
9. [IRCC — Federal Skilled Worker](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/who-can-apply/federal-skilled-workers.html) — including FSW 67-point selection criteria.
10. [IRCC — Federal Skilled Trades](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/who-can-apply/federal-skilled-trades.html).
11. [IRCC — Category-based selection](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/rounds-invitations/category-based-selection.html).
12. [IRCC — CRS calculator](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/check-score.html) — reference for parity testing, not a substitute for eligibility checks.
13. [IRCC — Canadian education additional CRS points](https://www.canada.ca/en/immigration-refugees-citizenship/services/immigrate-canada/express-entry/check-score.html) and applicable Ministerial Instructions / Immigration and Refugee Protection Regulations (verify precise credential-specific exceptions).

### Mandatory production-data follow-up before launch

The **numerical CRS scoring grids** and main effective-date rules above are included. Several regulatory-dependent data sets and edge-case validity decisions must **not** be invented or approximated. Before a production release:

- Import **complete test-specific raw-score → CLB/NCLC tables** (CELPIP, IELTS, PTE, TEF, TCF), with versions and boundary tests.
- Import **all effective-date high-wage NOCs**, not only software/IT examples.
- Import **all category-based NOC lists** with dates and Canada-only vs foreign-allowed period criteria.
- Verify the complete **Canadian education bonus eligibility exclusions** for distance learning/ESL/scholarship conditions.
- Confirm **FSW adaptability/arranged employment** subtleties and current **Nov 3 temporary public policy** terms for FSW/FST.
- Verify the mixed English CLB 4/5 boundary for the **25-point French bonus** against binding law rather than guessing.
- Validate exact **work-equivalency and partial-period** aggregation implementation against IRCC.
- Validate ECA expiration, language-test expiration, invitation-date age treatment, and FSW proof-of-funds exemptions against current official policy.
- Implement a provenance ledger and fail closed (`needs_review`) when authoritative rule tables cannot be fetched or the evidence is insufficient.

**Release gate:** Do not advertise this tool as *fully IRCC-equivalent* until these external data sets, regulatory edge cases and calculator parity tests are completed. Keep an internal legal/policy review process and schedule periodic IRCC change checks.
