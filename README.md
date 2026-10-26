# Cross-Benefit Duplicate Drug Billing Detection

**Same drug, Two bills and One payer.**

A buy-and-bill provider administers a specialty drug in-office and bills the **medical** benefit. The same drug clears the **pharmacy** benefit for the same member, on the same day or within days. Neither side knows what the other paid and the payer pays twice.

This pipeline detects that pattern across both benefits, scores each candidate pair, and produces a ranked investigator queue with quantified financial exposure.

---

## The Problem

Specialty drugs in *oncology, autoimmune, and ophthalmology* are frequently covered by two parallel benefit structures: the **pharmacy benefit** (adjudicated by a PBM) and the **medical benefit** (adjudicated by a medical payer or TPA). When a provider operates under a *buy-and-bill* model, acquiring the drug, administering it in-office, and billing the medical payer, the drug may also clear the pharmacy benefit for the same member.

Identifying these duplicates for classes of drugs is non-trivial for some reasons:

1. **Different code systems** - Pharmacy claims use NDC codes and Medical claims use J-codes (HCPCS). The same molecule maps to *different code* identifiers on each benefit side.
2. **Date misalignment** - Prescription fill date and administration date are rarely the same for a lot of specialty drugs and a strict date match eliminates real duplicates.
3. **No shared identifier** - No single claim field links a pharmacy record to a medical record. Member ID and drug identity must both be bridged across code systems.

A naive matcher that joins on `member_id + HCPCS code + date window` finds only part of true duplicates. It misses  biosimilar substitutions, Q-code billing gaps, and new formulation launches, exactly the scenarios with the highest per-unit cost.

---

## Approach

The pipeline runs in six sequential Databricks notebooks on Delta Lake. Below are the designed steps.

### 1. Molecule bridge (NB01 + NB04)

A CMS ASP crosswalk (`target_hcpcs_crosswalk`) maps every J-code and its biosimilar/Q-code variants to a canonical molecule name. Both pharmacy and medical claims are translated to molecule before matching.

The matching engine runs two passes, a naive and another one using a bridge:

| Pass | Join key | Window | Recall on synthetic data |
|------|----------|--------|--------------------------|
| Naive | `member_id + hcpcs_code` | 7 days | 53% (38 / 72) |
| Bridge | `member_id + molecule` | 7 days | 100% (72 / 72) |

The bridge surfaces 34 additional true positives that the naive pass misses including QCODE_GAP (18), NEW_FORM_CROSS (10), and DIFF_BIOSIM (8) scenarios are invisible without crosswalk translation.

### 2. Confidence scoring (NB04)

Each candidate pair receives a confidence score from two components, capped at 1.0:

- **Code alignment:** SAME_CODE +0.50, DIFF_CODE +0.30
- **Date proximity:** day 0 +0.50, days 1-3 +0.40, days 4-7 +0.25

### 3. Risk stratification (NB05)

Three signals are layered onto the confidence score to produce a composite risk score:

1. **NPI co-location (+0.15):** The prescriber NPI on the pharmacy claim equals the rendering NPI on the medical claim. Under a buy-and-bill pattern, the prescribing physician and the administering physician are often the same person. In the synthetic validation dataset, this signal fires for 100% of true positives and 12% of false positives.

2. **Dose concordance (+0.08):** Expected billing units for the pharmacy quantity (derived from CMS administered units and the HCPCS billing unit conversion) are compared to the medical claim's billed units. A concordance ratio between 0.5 and 2.0 indicates the two sides describe the same administered dose. TP median ratio: 1.011. FP median ratio: 0.062.

3. **Dual-cost flag (+0.10):** Combined cost (pharmacy + medical allowed) above the molecule median flags high-exposure pairs.

Risk tiers: HIGH >= 0.85, MEDIUM 0.65-0.84, LOW < 0.65.

### 4. Reporting layer (NB06)

Four output tables are written to `cross_benefit_audit`:

| Table | Rows | Primary use |
|-------|------|-------------|
| `member_risk_summary` | 104 | Case assignment by member exposure |
| `molecule_risk_summary` | 6 | Program management by drug class |
| `plan_risk_summary` | 135 | Payer / employer group exposure |
| `investigator_queue` | 232 | SIU referral list, ranked by risk score |

---

## Results

All figures are from the synthetic validation dataset (1,304 total claims, 72 seeded duplicate events).

| Metric | Value |
|--------|-------|
| Candidate pairs generated | 405 |
| True positives | 72 (100% recall) |
| Overall precision | 17.8% |
| HIGH-tier pairs | 232 |
| HIGH-tier precision | 31.0% |
| HIGH-tier recall | 100% |
| Pairs pruned by tier cutoff | 173 (43%) |
| TP financial exposure in queue | $652,712 |
| Total exposure across all pairs | $3,369,296 |

**Scenario coverage (all 5 scenario types recovered at 100%):**

| Scenario | Events seeded | Events found | Avg confidence |
|----------|--------------|--------------|----------------|
| EXACT_SAME_CODE | 20 | 20 | 1.000 |
| NEAR_DATE | 16 | 16 | 0.900 |
| QCODE_GAP | 18 | 18 | 0.761 |
| NEW_FORM_CROSS | 10 | 10 | 0.730 |
| DIFF_BIOSIM | 8 | 8 | 0.800 |

**Investigator efficiency:** Working only the HIGH-tier queue (232 of 405 pairs) reduces case volume by 43% while retaining 100% of the $652K confirmed duplicate exposure. Zero recall loss.

---

## Pipeline

| Notebook | Output table | Purpose |
|----------|-------------|---------|
| NB01 | `target_hcpcs_crosswalk` | CMS ASP crosswalk - molecule identity layer |
| NB02 | `pharmacy_claims` | Synthetic PBM claims (552 total, 72 seeded) |
| NB03 | `medical_claims` | Synthetic medical claims (752 total, 72 seeded) |
| NB04 | `candidate_duplicate_pairs` | Two-pass matching engine (405 pairs) |
| NB05 | `risk_scored_pairs` | NPI + dose concordance + cost risk stratification |
| NB06 | `member/molecule/plan summaries`, `investigator_queue` | Reporting layer |

---

## Tech Stack

- Apache Spark / PySpark (Databricks)
- Delta Lake
- Python 3 (pandas, matplotlib, numpy)
- CMS Average Sales Price (ASP) crosswalk
- Synthetic data with seeded duplicate events across 5 fraud scenario types

---

## Synthetic Data Design

Claims are generated with a dose-first methodology that uses `administered_hcpcs_units` value seeded into each duplicate event and propagated independently to both the pharmacy and medical sides with realistic noise. This ensures that the dose concordance signal is grounded in clinical plausibility rather than arbitrary quantities.

NPI co-location is seeded using a `md5(member_id) % 5` hash to assign buy-and-bill providers, the same hash produces the same provider assignment in both NB02 and NB03 without shared state between notebooks.

Six molecules across four therapeutic areas were selected: pembrolizumab, bevacizumab, ocrelizumab, infliximab, aflibercept and ranibizumab.

---

## Context

This project is part of a broader US payment integrity portfolio. It demonstrates cross-system data integration, clinical coding translation, and signal-based fraud detection on specialty drug claims, patterns directly applicable to SIU, payment integrity, and data engineering roles in managed care.
