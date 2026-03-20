# Advanced Risk & Portfolio Analytics — Dashboard Guide

Reference for the four Metabase dashboards covering Quikkred's credit risk lifecycle — from origination to early warning and collections.

---

## Table of Contents

1. [Dashboard Overview](#dashboard-overview)
2. [Vintage & Cohort Analysis](#1-vintage--cohort-analysis)
3. [Roll Rate & Transition Analysis](#2-roll-rate--transition-analysis)
4. [Risk Correlation Matrix](#3-risk-correlation-matrix)
5. [Early Warning System](#4-early-warning-system)
6. [Key Metrics Glossary](#key-metrics-glossary)
7. [How to Read These Dashboards Together](#how-to-read-these-dashboards-together)

---

## Dashboard Overview

| Dashboard | Primary Question |
|---|---|
| Vintage & Cohort Analysis | How are loans from a given disbursement week performing over time? |
| Roll Rate & Transition Analysis | How many loans are moving into deeper delinquency? |
| Risk Correlation Matrix | Which borrower segments and loan types carry the most risk? |
| Early Warning System | Which loans need collection attention today? |

---

## 1. Vintage & Cohort Analysis

**File:** `Vintage___Cohort_Analysis.pdf`

Tracks loan quality by origination week and month as loans age.

---

#### Loan Vintage by Disbursement Week (Status Breakdown)

Grouped bar chart (with secondary ₹ axis) showing per disbursement week:

| Series | What it tells you |
|---|---|
| Total Loans | Origination volume that week |
| Total Disbursed | Capital deployed |
| Total Outstanding | Unpaid balance remaining |
| Active Count | Open loans |
| Closed Count | Fully repaid loans |
| Overdue Count | Loans currently past due |
| Avg DPD | Average days past due |

The ratio of **Closed** to **Overdue** bars within a week shows repayment quality for that cohort. High Avg DPD on older cohorts is a red flag.

| Week | Total Loans | Total Disbursed | Closed | Overdue | Avg DPD |
|---|---|---|---|---|---|
| Jan 5 | 20 | — | 15 | 0 | 28.3 |
| Jan 12 | 163 | — | 110 | 49 | 21.8 |
| Jan 19 | 116 | ₹1.2M | 0 | 49 | 21.8 |
| Jan 26 | 151 | ₹768k | — | — | — |
| Feb 2 | 167 | ₹764.5k | 130 | 33 | 9.4 |
| Feb 9 | 241 | ~₹3M | 186 | — | — |
| Feb 16 | 221 | ₹2.4M | — | 142 | 4.8 |
| Feb 23 | 103 | — | — | — | — |

---

#### Weekly Cohort Overdue Rate Trend

Line + bar combo tracking cohort health over time.

| Series | What it tells you |
|---|---|
| Overdue Rate (green line) | % of cohort loans currently overdue |
| Closure Rate (orange line) | % of cohort loans fully repaid |
| Avg DPD (purple line) | Average days past due trend |
| Total Disbursed (yellow bars) | Volume reference |

If overdue rate and closure rate are both falling together, the active book is growing without performing or closing — a risk signal.

| Week | Closure Rate | Overdue Count | Avg DPD |
|---|---|---|---|
| Jan 5 | 75% | 5 | — |
| Jan 19 | — | 49 | 19.4 |
| Feb 2 | 77.8% | 33 | 6.5 |
| Feb 16 | 24.4% | 54 | 3.7 |

---

#### Vintage Overdue Rate By Count

Multi-line chart — count of loans per DPD bucket across disbursement weeks.

| DPD Bucket | Color |
|---|---|
| 0_Current | Teal |
| 1_DPD 1-7 | Pink |
| 2_DPD 8-15 | Dark purple |
| 3_DPD 16-30 | Green |
| 4_DPD 31-60 | Blue |
| 5_DPD 60+ | Light purple |

The 0_Current line should dominate. Rising 3_DPD or 4_DPD lines on recent cohorts = faster-than-expected aging into delinquency.

---

#### Vintage Overdue Rate By Outstanding Amount

Same structure as above but in ₹ outstanding. Surfaces high-value at-risk exposures — a single loan at ₹482.7k in DPD 31-60 matters more than ten loans in DPD 1-7.

---

#### Monthly Vintage Comparison (Outstanding Decay Curve)

Compares Jan vs Feb 2026 cohorts. A steeper downward slope in outstanding = faster repayment. A flat or rising outstanding line over time is a warning.

| Month | Total Loans | Total Disbursed | Outstanding | Avg Days Since |
|---|---|---|---|---|
| January 2026 | 450 | ₹5.1M | ₹3.0M | 57.7 |
| February 2026 | 732 | ₹8.3M | ₹4.1M | 49.8 |

Feb cohort is newer and larger — watch its decay curve over the next 4–6 weeks.

---

## 2. Roll Rate & Transition Analysis

**File:** `Roll_Rate___Transition_Analysis.pdf`

Tracks how loans move between delinquency states — recovering (roll back) or deteriorating (roll forward).

---

#### DPD Bucket Distribution (Active Portfolio)

Current snapshot of the active portfolio by DPD bucket.

| DPD Bucket | Loan Count | Outstanding | Avg Loan | Avg Bounce |
|---|---|---|---|---|
| 0_Current | 84 | ₹1.4M | ₹12.4k | 0 |
| 1_DPD 1-7 | 12 | — | ₹15.9k | 3.7 |
| 2_DPD 8-15 | 50 | — | ₹13.9k | 12.8 |
| 3_DPD 16-30 | 99 | ₹1.7M | ₹12.8k | 24.5 |
| 4_DPD 31-60 | 138 | ₹2.5M | ₹11.8k | 37.8 |
| 5_DPD 60+ | 2 | ₹64.5k | — | 0.9 |

Heaviest concentration is in **4_DPD 31-60 (138 loans, ₹2.5M)** — priority collection bucket. Rising Avg Bounce Count with DPD confirms systemic non-payment, not one-off misses.

---

#### Portfolio at Risk (PAR) Trend — Daily

PAR percentages for the current active book (385 loans).

| Metric | Value |
|---|---|
| PAR1 PCT | 79.81% |
| PAR7 PCT | 77.37% |
| PAR15 PCT | 66.66% |
| PAR30 PCT | 41.09% |
| PAR60 PCT | 3.15% |
| PAR90 PCT | 0% |

The gap between PAR1 (79.8%) and PAR30 (41.1%) means ~38% of the portfolio is in early-stage delinquency (1–30 DPD) — still recoverable. PAR60 at 3.15% is the truly impaired portion.

> ⚠️ PAR1 at ~80% is high. Validate the denominator and whether loans within a payment grace period are being counted as overdue.

---

#### Loans Entering Overdue This Week (Early Warning)

Table of loans that just crossed into overdue — the daily collections action list. Loans at DPD 6-7 are one step from rolling into 2_DPD 8-15 — call these first.

| Loan Number | Customer | Principal | Outstanding | DPD | Due Date |
|---|---|---|---|---|---|
| LN20261770818356961 | Tanuj Malik | ₹10,000 | ₹13,900 | 7 | Mar 13, 2026 |
| LN20261770807124114 | Sanjaykumar G. Shilu | ₹8,000 | ₹11,160 | 7 | Mar 13, 2026 |
| LN20261770828077643 | Md Umar Maqsood | ₹7,000 | ₹9,790 | 7 | Mar 13, 2026 |
| LN20261770787655636 | Bhallam V. Krishnamraju | ₹37,500 | ₹51,575 | 7 | Mar 13, 2026 |
| LN20261771948686187 | Soumava Basu | ₹15,000 | ₹18,750 | 6 | Mar 14, 2026 |

---

## 3. Risk Correlation Matrix

**File:** `Risk_Correlation_Matrix.pdf`

Correlates loan performance against borrower and loan attributes to identify high-risk segments.

---

#### CIBIL Score vs Loan Performance

| CIBIL Range | Total | Overdue Rate | Closure Rate | Avg DPD | Avg Amount |
|---|---|---|---|---|---|
| A: Below 600 | 4 | 25% | 75% | — | — |
| B: 600-650 | 102 | 48% | — | 20.8 | ₹14.5k |
| C: 650-700 | 136 | 38.2% | 56.6% | 14.8 | ₹13.6k |
| D: 700-750 | 204 | 38.2% | 59.8% | 15.2 | ₹13.9k |
| E: 750-800 | 149 | 37.6% | 60.4% | 13.7 | ₹11.3k |
| F: 800+ | 9 | 33.3% | 55.6% | — | ₹10.5k |

**600-650 band has the worst overdue rate (48%).** The 650-700 and 700-750 bands are nearly identical (~38%) — CIBIL is not providing meaningful differentiation in this mid-range. Consider tightening underwriting for the 600-650 band.

---

#### Multi-Dimensional Risk Matrix (Amount)

| Amount Group | Total | Overdue | Overdue Rate | Avg DPD |
|---|---|---|---|---|
| A: Small (<10K) | 369 | 97 | 234.1 | 110.7 |
| B: Medium (10-20K) | 483 | 145 | 330.7 | 132.1 |
| C: Large (20K+) | 258 | 50 | 235.3 | 79 |

Medium loans (₹10K-20K) carry the highest overdue rate — likely over-leveraged borrowers. Large loans perform better, possibly due to stricter underwriting at that ticket size.

---

#### Multi-Dimensional Risk Matrix (Tenure)

| Tenure | Total | Overdue | Overdue Rate | Avg DPD |
|---|---|---|---|---|
| 7 days | 172 | 30 | 263.2 | 39.3 |
| 15 days | 888 | 244 | 267.5 | 184.6 |
| 30 days | 50 | 18 | 269.4 | 97.9 |

Overdue rate is consistent across tenures (~265-270%) — tenure itself is not a strong risk predictor. **15-day tenure dominates by volume (888 loans)** and is where collection resources should be concentrated.

---

#### Multi-Dimensional Risk Matrix (CIBIL Group)

| CIBIL Group | Total | Overdue | Overdue Rate | Avg DPD |
|---|---|---|---|---|
| High (750+) | 155 | 60 | 301.1 | 101.7 |
| Medium (650-750) | 327 | 126 | 352.2 | 147 |
| Low (<650) | 628 | 106 | 146.8 | 73.1 |

Medium CIBIL (352.2) is the most problematic segment. Low CIBIL's lower overdue rate likely reflects smaller loan amounts and tighter terms given at underwriting.

---

#### Loan Amount vs Default Rate

| Amount Range | Total | Overdue Rate |
|---|---|---|
| A: 0-5K | 14 | 57.1% |
| B: 5-10K | 377 | 68.7% |
| C: 10-15K | 318 | 64.2% |
| D: 15-20K | 201 | 62.7% |
| E: 20-25K | 173 | 76.3% |
| F: 25-30K | 96 | 70.8% |
| G: 30K+ | 3 | 66.7% |

**₹20-25K has the highest default rate (76.3%).** The ₹5-10K band — highest by volume — also defaults at 68.7% and needs tighter screening.

---

#### Tenure vs Default Rate

| Tenure | Total Loans | Overdue | Closure Rate | Total Disbursed |
|---|---|---|---|---|
| 7 days | 51 | 18 | 64.7% | ₹354.5k |
| 15 days | 888 | 244 | 27.5% | ₹9.9M |
| 30 days | 172 | 30 | 17.4% | ₹2.1M |
| 60 days | 6 | 0 | 0% | ₹146.4k |
| 90 days | 2 | 0 | 0% | ₹58.9k |

30-day loans have a low closure rate (17.4%) — these are maturing and need proactive follow-up.

---

#### Employment Type vs Loan Performance

| Employment Type | Total | Overdue Rate | Closure Rate | Avg DPD | Avg Amount | Total Disbursed |
|---|---|---|---|---|---|---|
| Salaried | 1,060 | 25.8% | 67.4% | 9.2 | ₹13,009 | ₹12.1M |
| Self-Employed | 122 | 23.8% | 68% | 7.6 | ₹12,631 | ₹1.35M |

Self-employed borrowers slightly outperform salaried on overdue rate and Avg DPD — underwriting for this segment appears to be working.

---

## 4. Early Warning System

**File:** `Early_Warning_System.pdf`

Loan-level operational intelligence for collections teams.

---

#### Balance Monitor — Salary Detection & Early Warning

Table of loans with active bank consent, showing Loan Number, Loan Status, DPD, Consent Status, Snapshot Count, Latest Balance, Previous Balance.

| Flag | Interpretation |
|---|---|
| Negative latest balance + OVERDUE | Immediate escalation — near-zero self-cure probability |
| Balance dropping Previous → Latest | Cash flow deterioration |
| ACTIVE (DPD=0) + Negative balance | Pre-delinquency risk — intervene early |

---

#### Loans Due in Next 7 Days (Upcoming Maturities)

Table of 36 loans due by March 20, 2026, with customer contact details, principal, outstanding balance, and exact due datetime.

Contact borrowers 2–3 days before due date. Sort by Total Outstanding descending to prioritize high-value accounts.

---

#### E-Mandate Coverage vs Collection Success

| Mandate Status | Count | Overdue Count | Overdue Rate |
|---|---|---|---|
| Inactive Mandate | 210 | 76 | 22.1% |
| No Mandate | 972 | 663 | 68.2% |

**Loans with no e-mandate default at 3x the rate of those with an inactive mandate.** Activating e-mandates is one of the highest-leverage collection actions available.

---

#### Concentration Risk — Top 10 Customers by Outstanding

| Rank | Customer | Outstanding | Max DPD |
|---|---|---|---|
| 1 | Bhallam Venu Gopala Krishnamraju | ₹51,575 | 7 |
| 2 | Santosh Kumar | ₹42,400 | 48 |
| 3 | Bipin Singh | ₹41,900 | 53 |
| 4 | Mohinder Singh | ₹41,000 | 50 |
| 5 | Jithin Joy | ₹40,900 | 48 |
| 6 | Kautik Sonvane | ₹40,850 | 47 |
| 7 | Deepak Mishra | ₹40,678 | 58 |
| 8 | Ritesh D | ₹40,000 | 0 |
| 9 | Neha Omkar Nath Sharma | ₹40,000 | 38 |
| 10 | Satish Vasu Perambara | ₹39,700 | 37 |

Max DPD > 30 with outstanding > ₹40K = escalate to senior collections or legal notice. Deepak Mishra (2 loans, Max DPD 58) is a willful default risk.

---

## Key Metrics Glossary

| Term | Definition |
|---|---|
| **DPD** | Days Past Due — days since repayment was due but not received |
| **PAR** | Portfolio at Risk — % of outstanding portfolio overdue beyond a threshold (PAR30 = 30+ days) |
| **Overdue Rate** | % of loans in a cohort or segment currently overdue |
| **Closure Rate** | % of loans fully repaid and closed |
| **Avg DPD** | Average days past due across overdue loans in a group |
| **Vintage** | A cohort of loans grouped by disbursement date (week or month) |
| **Roll Rate** | Rate at which loans move between DPD buckets (forward = worsening, backward = recovery) |
| **E-Mandate** | Electronic mandate for automatic debit from borrower's bank account on due date |
| **Bounce Count** | Number of times an auto-debit / NACH instruction has failed |
| **Concentration Risk** | Overexposure to a single borrower, geography, or segment |
| **Outstanding Decay Curve** | How a cohort's outstanding balance decreases over time as loans are repaid |

---

## How to Read These Dashboards Together

```
Vintage & Cohort Analysis
  → Spot cohorts with rising overdue rates or slow outstanding decay

Roll Rate & Transition Analysis
  → Identify which DPD buckets are growing and how fast loans are rolling forward

Risk Correlation Matrix
  → Attribute delinquency to specific borrower segments, loan amounts, or tenures

Early Warning System
  → Act on specific loan numbers and customer contacts today
```

### Weekly Review Checklist

- [ ] Any new cohorts showing overdue rate > 20% within 2 weeks of disbursement?
- [ ] Is the DPD 31-60 bucket growing week-over-week?
- [ ] Is PAR30 above 40%? Which CIBIL / amount segment is driving it?
- [ ] How many loans are due in the next 7 days? Have borrowers been contacted?
- [ ] Any balance monitor loans showing newly negative balances?
- [ ] What % of the active portfolio has no e-mandate?
- [ ] Any borrower appearing in both the concentration risk table and early warning list?

---

*Built on MongoDB aggregation pipelines, served via Metabase. Data reflects Quikkred NBFC portfolio — Satsai Finlease Private Limited.*
