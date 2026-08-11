# The fact models

Eight facts, in three families. Each section gives the grain, what the model exists to answer, the columns
that matter, and the ways it gets misread.

**Grain is the first line of every section.** It determines whether a column can be summed without
double-counting, and it is the fastest way to tell whether a model answers the question being asked.

| Family | Model | One row per |
|---|---|---|
| Premium & earning | [`fact_pds_policy_transaction_premium`](#fact_pds_policy_transaction_premium) | policy transaction |
| | [`fact_pds_premium_earned_monthly`](#fact_pds_premium_earned_monthly) | transaction × month-end |
| Exposure & rate | [`fact_pds_policy_exposure_detail`](#fact_pds_policy_exposure_detail) | transaction × exposure basis |
| | [`fact_pds_policy_exposure_rate`](#fact_pds_policy_exposure_rate) | policy transaction |
| | [`fact_pds_policy_rate_component`](#fact_pds_policy_rate_component) | rated premium row |
| | [`fact_pds_composite_group_rate`](#fact_pds_composite_group_rate) | transaction × negotiated group |
| Reporting surface | [`fact_pds_policy_term`](#fact_pds_policy_term) | policy term |
| | [`fact_pds_policy_renewal_rate_change`](#fact_pds_policy_renewal_rate_change) | renewal pair |

Supported by **`dim_insured`** — one row per customer, spanning both systems.

[start_here.md](start_here.md) covers the three decisions these models encode, and
[glossary.md](glossary.md) the vocabulary. Design rationale sits at the [bottom of this
file](#design-notes) rather than the top, since it is not needed to query the models correctly.

---

## Before you join anything

**These are not a star schema, and most of them are not meant to be joined to each other.** They are one
lineage published at the grains different questions need, so the same premium appears in several of them at
different shapes. That makes the verdict table below the operative reference rather than an entity diagram.

### Join reference

| Join | Verdict |
|---|---|
| `fact_pds_policy_term` → `dim_insured` on `conformed_insured_key` | Yes — the intended dimension join |
| `fact_pds_policy_renewal_rate_change` → `fact_pds_policy_term` on either key | Yes, many-to-one, for attributes only |
| `fact_pds_policy_exposure_detail` → `fact_pds_policy_exposure_rate` on `policy_transaction_id` | Yes, drill-down |
| `fact_pds_policy_rate_component` → `fact_pds_policy_exposure_rate` on `policy_transaction_id` | Yes, drill-down |
| `fact_pds_composite_group_rate` → `fact_pds_policy_rate_component` on the group number | Yes — the documented route |
| `fact_pds_policy_term` → `fact_pds_policy_exposure_rate` on `policy_key` | Only with a `transaction_type` filter. A term has many transactions; filtering to `'Issue'` holds it one-to-one. Without that filter every policy multiplies by its transaction count. |
| `fact_pds_policy_transaction_premium` → `fact_pds_premium_earned_monthly` | **No** — fans out by month |
| `fact_pds_policy_term` → `fact_pds_policy_transaction_premium` for measures | **No** — the term already aggregates it. Drill down to list a term's transactions, but never re-sum. |
| `fact_pds_policy_exposure_detail` → `fact_pds_policy_rate_component` | **No** — no shared grain |
| `fact_pds_policy_transaction_premium` → `fact_pds_policy_exposure_rate` | 1:1 on PDS transactions, but the rate fact omits voided policies, so an inner join silently drops premium. Left join only, and do not mix written with annual measures in one total. |

Two shortcuts worth knowing, because they avoid a join entirely: `fact_pds_premium_earned_monthly` carries
`policy_eff_date_key`, so earned premium rolls up to term, program or subsidiary without touching
`fact_pds_policy_term`. And `fact_pds_policy_term` already carries the rate and ladder, so most rate
reporting never needs `fact_pds_policy_exposure_rate` at all.

---

## Premium measures in these models

The written / annual / earned distinction is defined in
[start_here.md](start_here.md#3-three-things-these-models-decide-for-you). What matters when reading the
facts below is that the column name is the only marker: `policy_premium` and `incremental_premium` are
written; `rated_premium`, `policy_premium_annual` and `underlying_premium` are annual; `earned_premium` and
`unearned_premium` are earned.

---

## `fact_pds_policy_transaction_premium`

**Grain.** One row per policy transaction, PDS and Velocity unioned. Key: `transaction_premium_key`.
Roughly 22,000 rows.

**Answers.** "What premium did each transaction book, and over what window does it earn?" This is the
single definition of written premium and of premium earning for both systems.

**How earning works here.** The model publishes the *parameters* of earning — a start date, an end date, a
day count and a daily amount — rather than a row per period. Earned premium at any date is then a closed
form: days elapsed × daily amount. That is why it can answer any valuation date, including mid-month,
without a date spine.

Two properties make it correct where the older policy-level approach was not. The earning window is **per
transaction, not per term**, so a mid-term endorsement earns from when it was effective rather than from
policy inception. And the window is **truncated by cancellation**, so a cancelled policy stops earning.

**Key columns.**

- *Two date axes.* `transaction_effective_date` — when cover changed. `transaction_booked_date` — when the
  system recorded it. Financial reporting needs both, and they are frequently months apart.
- *Premium and its split.* `incremental_premium` is the booked increment. Part of a transaction can be
  **fully earned at once** rather than pro-rata (minimum earned premium, some fees), so it splits into
  `incremental_premium_fully_earned` and `incremental_premium_pro_rata`.
- *Earning parameters.* `earn_start_date`, `earn_end_date`, `earn_days`, `daily_earn_amount`.
- *An audit alternative.* Audit premium can either be recognised when booked or spread across the term it
  audits. Both treatments are published (`audit_earn_start_date` / `audit_earn_end_date`); non-audit rows
  carry the two identically.
- *Quality claims.* `earning_precision` — `transaction_level` or `term_level`; every Velocity row is
  term-level. `effective_date_source` — how the effective date was obtained, which is `system` on PDS and
  derived on Velocity.

**Traps.**

- `daily_earn_amount` is a rate, not money. Never sum it.
- The fully-earned and pro-rata columns are **components** of `incremental_premium`. Adding any two of the
  three double-counts.
- Pick one earning treatment — standard or audit — for a whole report. Mixing them double-counts audit
  premium.
- `source_cumulative_written_premium` is the source's own **cumulative** figure, for reconciliation only.
  Never sum it alongside the increments, and never earn against it. It disagrees with the sum of
  increments on a handful of terms in both directions, so neither is universally authoritative.
- **Cancellations need no special handling.** Booked increments over their own windows already inherit
  whatever the underwriter did — pro-rata, short-rate, minimum earned. Do not add short-rate logic.
- Velocity audit premium is recognised at term expiry and audits can land up to three years later, so a
  legacy month already reported **can move**.

Held by `assert_earned_plus_unearned_equals_written`, `assert_cancelled_terms_are_fully_earned` and
`assert_transaction_premium_excludes_tax_surcharge`.

---

## `fact_pds_premium_earned_monthly`

**Grain.** One row per transaction × month-end **within that transaction's own earning window**. Roughly
230,000 rows — against about 9.2M in the model it is intended to replace.

**Answers.** "What earned, unearned and written premium do we show for each accounting month?" It exists
only so that BI tools which cannot express the closed form get a row per period. It is strictly derived —
if it ever disagrees with `fact_pds_policy_transaction_premium`, this model is wrong.

**Key columns.** `earned_premium_to_date` (cumulative), `earned_premium_in_month` (the period movement),
`unearned_premium`, and `written_premium_to_date` — which is the one measure in the project genuinely
evaluated at a **past** date, counting a transaction only if it was *booked* by that month-end.

**Traps.**

- **Month-ends only.** It cannot answer a mid-month valuation date; read the closed form for that.
- **Every `_to_date` column is stated as at its row's month-end and must not be summed across months.**
  That applies to `written_premium_to_date` exactly as it does to `earned_premium_to_date`: a transaction's
  figure is repeated on every month-end in its earning window, ten times over on average, so summing
  either across months inflates it about tenfold. Use `earned_premium_in_month` for a period figure, or a
  single month's rows for a balance.
- Do not mix the two earned columns in one total.
- **There is no written-premium movement column, and one cannot be added at this grain.** A transaction is
  written once, and no row here is unique to the month it was written in. For written premium by month read
  `fact_pds_policy_transaction_premium` grouped on `last_day(transaction_booked_date)`. Do not instead
  filter this fact to each transaction's booked month-end — a transaction booked outside its own earning
  window has no such row and drops out silently.
- The spine is bounded at **both** ends, so a transaction is **absent** from months after its window
  closes. Absence means earned premium is flat at its final value, not zero. A report that left-joins a
  full calendar will show zeros where it should show a flat line.

Held by `assert_monthly_earned_matches_closed_form`, which recomputes every row independently.

---

## `fact_pds_policy_exposure_detail`

**Grain.** One row per transaction × exposure basis × divisor. Deliberately long rather than wide, so a
new exposure basis arrives as a *row* — RSI alone has around twenty of them.

**Answers.** "What is this policy rated on, how much of it is there, and what rate did each basis get?"
It is the extensible truth beneath the policy-level rate, and the only place a mixed-basis policy's rate
is still readable per basis.

**Key columns.** `exposure_basis` and `basis_type`; `exposure_amount`; `rated_premium` (annual);
`rate` and `rate_unit`; `rate_comparison_group`. Both the basis and the divisor carry a `*_source` column
recording where they came from, including where they were derived rather than read.

**Traps.**

- **`exposure_amount` must not be summed across bases on one policy** — square feet and apartment units
  are not the same unit. Summing across policies within one comparison group is fine.
- Only the coverages belonging to the resolved rating method contribute. This is the single most important
  filter in the model — see idea 2 in the [README](README.md).
- **Not every transaction appears.** Policies flat-cancelled back to inception with nothing retained are
  absent, along with their issuing transactions. `int_pds_policy_rating_method` holds the excluded population.
- `rate_percentile` and `is_rate_outlier` are **descriptive only** — they surface bad *inputs*, such as an
  exposure keyed with the wrong number of zeros. Never filter a report on them and never treat a flag as
  an error. Ties are load-bearing: a filed rate shared by dozens of policies correctly does not flag.

---

## `fact_pds_policy_exposure_rate`

**Grain.** One row per policy transaction, PDS only. Key: `policy_transaction_id`. This is the rate fact.

**Answers.** "What rate did we charge on this transaction, per unit of what, and how much of the price
does that rate actually account for?" Plus the full ladder — charged, manual and normalized rates.

**Key columns, by theme.**

- *Method.* `rating_method`, `exposure_strategy`.
- *The published rate.* `rate`, `rate_unit`, `exposure_amount`, `exposure_basis`, `basis_type`,
  `rate_divisor`, `rate_comparison_group`, `rate_is_reportable`, `rate_absence_reason`.
- *How the basis was resolved.* `basis_status`, `n_bases`, `dominant_basis_premium_share`.
- *How much of the money the rate covers.* `rated_premium_share` (the published numerator as a share of
  annual **policy** premium, and the input to the dominance reportability bar), `premium_outside_rate`
  (rating premium the rate does not cover — read this first when a rate looks low), `premium_leakage`
  (rating premium minus the exposure components, deliberately not netted so a gap cannot hide),
  `premium_identity_holds`. Premium is decomposed into named components — class code,
  composite, liquor, flat charge, manual adjustment, tax and surcharge — so that **every dollar of policy
  premium is either in the rate or in a named bucket explaining why it is not.**
- *How close a value sits to a gate.* `terms_coverage_share` (how much of the rated premium carries a
  divisible terms factor — below the cutoff the normalized rate is withheld), `manual_premium_share`
  (**signed**: negative means the rate reads high), `is_manual_premium_material`. Each gate's number is in
  [publishing_a_rate.md](publishing_a_rate.md).
- *The ladder.* `base_rate`, `manual_rate`, `normalized_rate`, `terms_factor`,
  `underwriting_modification_factor`, `normalization_basis`, and an absence reason for each rung.
- *Program-specific.* Senior living: `derived_schedule_mod`, `n_schedule_mod_locations`. Excess/umbrella:
  `underlying_match_status`, `underlying_premium`, `pct_of_underlying`. Auto: `vehicle_count`.

**Traps.**

- **Do not filter out rows with no rate.** Read `rate_absence_reason`; filter `rate_is_reportable` when you
  need the rate itself, but keep the rows in premium totals. Every reason string is catalogued in
  [publishing_a_rate.md](publishing_a_rate.md).
- **A published rate does not imply a published ladder.** The rungs are gated separately, so a policy can
  carry a good charged rate and withhold its normalized rate with its own stated reason.
- **Never average or compare rates outside `rate_comparison_group`, or normalized rates outside
  `normalization_basis`.**
- `terms_factor` and `underwriting_modification_factor` are **premium-weighted harmonic means**. Do not
  average them, and do not expect them to match a simple average of the underlying rated rows. Both are
  **withheld** on a multiple-composite policy — the per-group figures live in `fact_pds_composite_group_rate`.
- Two columns are **memo only and not additive**: `overridden_class_code_premium` (which runs orders of
  magnitude above real premium) and `liquor_premium` (a subset of class-code premium, not an addition).
- `base_rate` must be read **with** `manual_rate`, never against the charged rate alone. On senior living
  every factor between them reduces the price, so base-versus-charged suggests a discount where the risk
  was actually loaded.
- `derived_schedule_mod` is derived, is a residual, and summarises several separate per-location
  judgements. Read it with `n_schedule_mod_locations`, judge plausibility per location in
  `int_pds_kbsi_schedule_mod`, and never present it as a source value.
- On excess rows `underlying_premium` duplicates `exposure_amount` and `pct_of_underlying` duplicates
  `rate` — they are aliases for readability, not extra measures.
- **Cancellations are reportable here** and are judged on the same terms as anything else. Do not
  reintroduce a cancellation filter.

Held by `assert_premium_identity_holds`, `assert_rated_premium_is_complete`,
`assert_rate_contract_is_consistent`, `assert_ladder_absence_reasons_are_exhaustive` and the four
per-program worked-example tests.

---

## `fact_pds_policy_rate_component`

**Grain.** One row per **rated premium row** — a coverage on the class-code programs, a location on senior
living, a vehicle-coverage line on auto.

Premium-row rather than coverage grain is deliberate: a GL coverage can carry both premises/operations and
products premium, each with its own limit factor and deductible credit, so a coverage-grain model could
hold neither.

**Answers.** "*Why* did the rate move?" The policy-level rate says whether price moved; only this grain
says which coverage or location moved it, and through which factor.

**Key columns.** `rate_line` — read this first, always. Then the chain: `bureau_loss_cost`,
`loss_cost_multiplier`, `base_rate`; what the customer bought (`increased_limit_factor`,
`deductible_credit`, plus program-specific territory, claims-made and corridor factors); what the
underwriter decided (`rate_modification_factor`); and the derived `row_achieved_rate`, `manual_rate` and
`normalized_rate`.

**Traps.**

- **The same column means different things on different lines.** This is the largest trap in the model.
  `deductible_credit` is a fraction **subtracted** from the limit factor on GL and a **multiplier** on
  senior living. `increased_limit_factor` is a **load** on GL, liquor and auto and a **reduction** on
  senior living. Always read `rate_line` first.
- A factor a line does not carry is **NULL, never a substituted 1.0**. But a factor that *is* recorded at
  exactly 1.0 is published as 1.0 — a disclosed neutral value is a statement of fact, where a NULL would
  invite someone to go looking for missing data.
- `row_exposure_amount` is the row's **own** exposure and will not sum to the policy-level
  `exposure_amount`. It is not meant to.
- The rates here **do not sum or average to** the transaction-grain rungs.
- `final_modified_rate` is a rate, not a factor. Never multiply it into anything.
- Products/completed-operations rows carry very small premiums, so they reconcile on an **absolute**
  tolerance. Do not add a percentage test across all premium types.
- `has_unverified_terms_component` and `has_ambiguous_factor` are descriptive flags, not error conditions.

---

## `fact_pds_composite_group_rate`

**Grain.** One row per transaction per composite group. Contractors only, by construction.

**Answers.** "On a policy with several negotiated groups of class codes, what rate did each group get?"
These policies have no single policy-level rate, so `fact_pds_policy_exposure_rate` withholds one and names
this model in its absence reason.

**Traps.**

- **`build_up_premium` must never be summed into premium or compared against charged premium.** It is far
  larger, by a varying multiple, because the rating engine summed every build-up row it held including
  templated duplicates. It is published only so the terms factor's weighting can be audited.
- Those duplicates are **deliberately not de-duplicated** — the factor is a ratio and therefore invariant
  to the scale of the weights.
- `terms_factor` is a premium-weighted harmonic mean; **never average it across groups** to make a policy
  figure. Preventing exactly that is why this model exists as a separate fact rather than as columns
  repeated onto the rows beneath it.
- `manual_rate` is NULL **by nature** — on a composite policy the negotiated rate *is* the judgement.
- The rate is **withheld** where it falls outside a plausible band — a fixed pair of bounds, not one derived
  per program; a few groups carry an
  exposure of a cent or two against a whole-dollar premium, which yields an absurd rate. That guard bites
  at group grain where it does not at policy grain, because aggregation dilutes such a group away.
- Join it to `fact_pds_policy_rate_component` on the group number, which appears on both.

---

## `fact_pds_policy_term`

**Grain.** One row per policy term, both sources. Key: `policy_eff_date_key`. Roughly 6,300 terms.

**Answers.** "Give me a policy listing with premium, exposure and rate, with no joins." This is the BI
handoff and the model most reports should start from.

### One rate, valued at a stated date

**The term carries a single rate.** It is the price per unit of exposure implied by everything booked on
the term so far, including any audit that has landed. It **moves** as audits and endorsements land, so
`rate_as_of` is published beside it and is not optional — a stale extract must identify itself rather than
implying it is current.

Everything that gives the rate meaning is read off the same transaction as the rate itself: `exposure_amount`,
`exposure_basis`, `rate_unit`, `basis_status`, `rate_comparison_group`, and the ladder that decomposes it.
Splitting the rate from its own decomposition is how a terms factor ends up dividing out something the
price never contained.

**Terms carrying landed audits are compared directly against terms that have not been audited yet.** That is
deliberate: the audit is part of what the policy cost. `development_age_months` states how mature each term
is, so you can see the maturity you are comparing at and impose your own floor if you want one.

**Key columns.** The identity and attribute block — policy number, `conformed_insured_key`, subsidiary,
program, program year, broker, underwriter, named insured, address, renewal flag. Term and cancellation
dates. Limits and retentions. Then `written_premium`, `earned_premium`, `exposure_amount` and `rate`, plus
the ladder (`manual_rate`, `base_rate`, `normalized_rate`, `terms_factor`) with an absence reason for the
manual and normalized rungs.

**Traps.**

- **There is no charged-rate absence reason on this model** — only `manual_rate_absence_reason` and
  `normalized_rate_absence_reason`. When `rate` is NULL, this model tells you *that* through
  `rate_is_reportable` but not *why*. For the plain-words reason, join to `fact_pds_policy_exposure_rate`
  on `policy_key` — and note it resolves for PDS terms only, since the rate fact holds no legacy
  transactions.
- `rate_as_of` is **always the build date** — the model is not back-datable today. For a knowledge-axis
  figure at a past date use `fact_pds_premium_earned_monthly`.
- `written_premium` and `written_premium_all_transactions` are **overlapping, not additive**. They differ
  only where premium was booked after the as-of date, which today is one term in the book. Never add them.
- `sir` and `deductible` are the **raw source strings**, kept unchanged for drop-in parity with existing
  consumers; `sir_amount` and `deductible_amount` are the parsed numbers.
- A `retention_amount` of 0 is a **real first-dollar retention**; NULL means none is recorded. A mixed
  retention basis fills neither amount column. `corridor_retention_amount` is a separate layer and is
  never summed into the retention.
- `conformed_insured_key` joins to **`dim_insured` only** — never to `dim_underwriting_insured` or
  `dim_suremga_insured`, which carry a same-named key over a different population.

---

## `fact_pds_policy_renewal_rate_change`

**Grain.** One row per customer × line of business × current term that has a predecessor. Key:
`renewal_pair_key`. **Covers general liability on Contractors, Habitational and Sports & Entertainment;
Business Auto and Excess on every program that writes them; and Healthcare Professional Liability on Senior
Living (the KBSI book).**

**Answers.** "Did we get rate?" — meaning is this customer paying more per unit than last year, which is a
different question from whether one policy is priced above another.

**Key columns.** The pair identity (`pair_source` — whether the source named the predecessor or the pairing
was derived from consecutive terms, and the two are not equally trustworthy). Then the comparison at
inception: `rate_change_pct`, `exposure_change_pct`, `premium_change_pct`, `basis_comparable`. Then the part
that separates price from coverage: `terms_factor_change_pct`, **`normalized_rate_change_pct`** and
`terms_changed_at_renewal`. And the gate: `rate_change_is_reportable` with
`rate_change_absence_reason`.

`normalized_rate_change_pct` is the closest thing to "did we actually get rate", because it is the change
after holding limits and deductible constant.

**Traps.**

- **Read `rate_change_is_reportable` before reading `rate_change_pct`.** Comparability is a gate, not a
  caveat — a payroll-rated term against a receipts-rated term produces arithmetic, not a price change.
- **What trips that gate is different in each book, and knowing only one book's story will mislead you.**
  On Contractors it is the rating method, identified by the exposure basis: a class-code-rated term against
  a composite-rated one is refused. On Habitational, almost nothing — the book is rated per door throughout.
  On Sports & Entertainment it is the exposure unit itself, which that book changes at renewal far more
  often than the others; roughly one pair in nine is refused for it. Per program in
  [programs.md](programs.md).
- **The lines beyond general liability refuse nothing on comparability**, because each is rated on one unit
  that does not change between a customer's two terms. Business Auto is per vehicle (not normalized, so no
  terms-adjusted change). Excess is a percentage of the underlying primary premium (also not normalized);
  the Contractors book splits into two attachment towers, `Excess 3X2` and `Excess 5X5`, which share one
  comparison group but never pair against each other because a term pairs only within its own line. Senior
  Living Healthcare Professional Liability (KBSI) is per skilled bed and does carry a terms-adjusted change,
  but its normalization restates rates *up* where the general liability books restate *down* — so a senior
  living normalized rate must never sit beside a general liability one, which the shared-basis gate on
  `normalized_rate_change_pct` enforces.
- **One Sports & Entertainment comparison is allowed on our judgement rather than the source's.** That
  program uses several interchangeable names for the same headcount, so a policy's label can move between
  its own two terms while the count does not. Those pairs are published and flagged with
  `basis_comparable_by_renamed_unit`; exclude them if you want only pairs the source itself makes
  comparable. The judgement rests on the recorded quantity holding still — `exposure_change_pct` is the
  number the rule turns on, and the threshold is in [publishing_a_rate.md](publishing_a_rate.md). It is
  deliberately a pair-level allowance and **not** a widening of `rate_comparison_group`: two terms of one
  policy whose label moved while the count held still are almost certainly the same unit, whereas two
  different customers rated per member and per admission are not.
- **Do not average or sum `rate_change_pct`** without a stated segment and date window. This table
  publishes pairs; a book-level average is a property of a report's scope, not of this table. And a book
  average will mostly be mix: the whole-book median rate can fall a fifth while the same accounts renew
  up a few percent, because new business prices lower.
- `normalized_rate_change_pct` is refused unless the two sides share a normalization basis, and is
  available on far fewer pairs than the raw change — because most pairs still have a pre-cutover term on
  one side. That is a **population** limit, not a defect.
- `has_coverage_gap` and `has_short_interval` are flags, not exclusions. Whether a cancel-and-rewrite is a
  renewal is an underwriting question, not a modelling one.
- Each side is that term's **single rate**, so a pair reads everything booked on both terms. A term whose
  audit has landed can be compared against one whose has not; both development ages are published so you
  can see the maturity you are comparing at.

---

## `dim_insured`

**Grain.** One row per conformed customer. Key: **`conformed_insured_key`**.

The two systems key customers differently and in separate namespaces. `dim_insured` bridges them using
**the predecessor policy number the source itself records on each policy** — so every bridge traces to a
link the business made, not to a name match.

There is **no fuzzy matching, no name normalisation and no tax-id matching anywhere in the key.** The
named insured is a display attribute only and must never be joined on: it is free text holding a policy's
whole schedule of affiliates, re-entered differently on each transaction, so exact matching on it fails
silently on exactly the largest accounts.

Where a policy number resolves to more than one customer the bridge is **refused and labelled**, never
resolved by picking one — a wrong bridge merges two customers' rate histories and publishes a rate change
for a customer who never renewed.

`bridge_status` and `bridge_absence_reason` explain each unbridged customer. Most are not defects: a
customer who left before the migration has no successor, and new business has no predecessor.

---

## Design notes

Why the models are shaped this way. You do not need any of this to query them correctly — it is here for
whoever extends them next.

### One lineage, reshaped

Each downstream fact is built *from* the one above it rather than designed to sit beside it:

```
fact_pds_policy_transaction_premium ──┬──→ fact_pds_premium_earned_monthly
                                  │
                                  └──→ fact_pds_policy_term ──→ fact_pds_policy_renewal_rate_change
fact_pds_policy_exposure_rate ────────────↗
        ↑
fact_pds_policy_exposure_detail
```

The reason is that the hard decisions in this domain are all **choices between measures that look
interchangeable and are not** — written against annual premium, one valuation date against another, which
of a policy's many transactions carries "the" rate. Each fact makes those choices once, so that a report
inherits them instead of making them. A join that reaches around a fact usually reaches around a decision
with it.

That gives three kinds of relationship between facts, with different rules:

**1. Derived reshaping — choose one, never join.** The downstream model contains the *same* premium as the
upstream one, exploded or collapsed. It publishes no measure of its own, and a test recomputes it against
its parent. Joining them repeats premium.

**2. Drill-down — join freely, many-to-one.** A finer-grained fact keyed by something the coarser one
carries. Safe, because you are joining *down* to explanatory detail rather than adding measures.

**3. Different branch — do not join.** Two facts below the same parent at unrelated grains. They share a
transaction id and nothing else, so a join cross-products inside each transaction.

### The two derived pairs

**`fact_pds_premium_earned_monthly` is `fact_pds_policy_transaction_premium` exploded into periods.** The
transaction fact publishes the *parameters* of earning — start date, end date, daily amount, and the
component that earns whole at once — so earned premium at any date is arithmetic. The monthly fact
evaluates those parameters at each month-end and holds no premium of its own;
`assert_monthly_earned_matches_closed_form` recomputes every row, so if the two disagree the monthly
model is the one that is wrong. Choose between them: the monthly fact for a row per accounting period,
the transaction fact for any date that is not a month-end. Joining them would repeat each transaction's
premium once per month in its window.

**`fact_pds_policy_renewal_rate_change` is a self-join of `fact_pds_policy_term`, already performed.** It reads
the term fact twice, pairs each term with its predecessor, and publishes both sides as `current_*` and
`prior_*` columns — so the pairing rules and the comparability gate live in the model rather than in every
report that wants a rate change.

Here a join back **is** supported and tested: both `current_policy_eff_date_key` and
`prior_policy_eff_date_key` have declared relationships to `fact_pds_policy_term.policy_eff_date_key`. Use it
to pick up term attributes the pair does not carry — broker, underwriter, cancellation status. What you
must not do is sum term premium across it, because a term appears in up to two pairs: once as the current
term, and again as the prior term of the following year.

### Why `fact_pds_policy_term` is the reporting surface

It exists to settle three things once, in the model, that a report author would otherwise have to settle
per report — and would mostly get wrong:

1. **Which transaction's rate is "the" rate.** A policy has many transactions, each with its own rate. The
   term fact picks one — the latest transaction carrying a reportable rate — and reads the exposure, unit,
   comparison group and the whole ladder off that same transaction. Anyone joining the rate fact themselves
   has to make that choice, usually implicitly, and splitting the rate from its own decomposition is how a
   terms factor ends up dividing out something the price never contained.
2. **What "as of" means.** The rate moves as audits and endorsements land, so `rate_as_of` is published
   beside it rather than left implicit.
3. **Which measures may share a total.** Written, annual and earned premium all appear, named so they
   cannot be casually added together.

Plus the plain reason: it is denormalised, so a policy listing needs no joins at all and a BI tool can be
pointed straight at it.

---

## Relationship to existing models

**Nothing has been retired.** `base_rpt_policy`, `base_rpt_policy_insurity`, `base_rpt_policy_velocity`,
`base_rpt_written_premium_*` and `rpt_written_premium` are untouched and still serve their consumers,
including a production Streamlit app that computes a loss ratio from `base_rpt_policy.earned_premium`.
Reshaping that in place would have moved a live number with no test to catch it.

What is intended to be superseded, when consumers are migrated:

- **`base_rpt_policy`'s earning.** Its spine has no per-policy upper bound and earns cumulative written
  premium from policy inception, so a mid-term endorsement earns as though it had been present on day one
  and a cancellation never truncates. The net error is under a percent, but it is concentrated — rare and
  large rather than diffuse. `warn_fact_pds_policy_term_reconciles_to_base_rpt_policy` runs the two side by
  side and requires every difference to have a named cause.
- **Two layer-violating reads.** `dim_claim` and `int_tpm_rsi_claim_deductible_logic` currently read
  reporting models. The limit, SIR and deductible columns on `fact_pds_policy_term` exist so they can be
  rewired, and were verified equal on every term.

One model on this branch is **diagnostic only and must stay unconsumed**: `stg_pds_lob_factor`. Reading it
alongside the premium-grain judgement factor would multiply the underwriter's load in twice and look
entirely plausible. `assert_judgement_factor_reads_premium_grain_only` enforces that.

---

## Never sum, never average

| Column | Why |
|---|---|
| `daily_earn_amount` | A rate, not money. |
| `earned_premium_to_date` | Cumulative. Use `earned_premium_in_month`. |
| `incremental_premium_fully_earned` / `_pro_rata` | Components of `incremental_premium`. |
| `written_premium` / `written_premium_all_transactions` | Overlapping subsets of one another. |
| `source_cumulative_written_premium` | Cumulative snapshot, reconciliation only. |
| `exposure_amount` across bases on one policy | Different units. |
| `rate`, `normalized_rate` across comparison groups or normalization bases | Not comparable. |
| `terms_factor`, `underwriting_modification_factor` | Premium-weighted harmonic means. |
| `build_up_premium` | Not premium — includes templated duplicates. |
| `overridden_class_code_premium` | Memo only; orders of magnitude above real premium. |
| `liquor_premium` | A memo subset of class-code premium. |
| `rate_change_pct` | Publish per pair; a book average is mostly mix. |
| `derived_schedule_mod` | A derived residual summarising several location judgements. |
