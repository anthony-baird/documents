# PDS reporting foundation

This folder documents the fact models built on the Policy Data Store (PDS) — what they contain, how the
programs they describe are priced, and how to use them without getting a wrong number.

## Read in this order

The first two rows cover ordinary use. The rest is reference.

| # | Read this | For |
|---|---|---|
| 1 | [start_here.md](start_here.md) | Where the tables are, which one to query, and the three decisions the models encode |
| 2 | [glossary.md](glossary.md) | Every insurance and modelling word the other files assume you know |
| 3 | [publishing_a_rate.md](publishing_a_rate.md) | What a published rate claims, how it is decided, and every threshold and absence reason — read before writing a rate query |
| 4 | [analytics_examples.md](analytics_examples.md) | Ten worked questions, each with the mistake that makes the answer wrong |
| 5 | [fact_models.md](fact_models.md) | The eight fact models — grain, columns, and which joins are safe |
| 6 | [programs.md](programs.md) | How each program is actually priced, and the one trap in each |
| 7 | [relational_model.md](relational_model.md) | The vendor's own raw PDS table model — read before writing `stg_pds_*` SQL |
| 8 | [extending.md](extending.md) | Adding a program, a column, or a new exposure unit without breaking the contract |
| 9 | [changelog/](changelog/README.md) | Fixes to these models, one file per change — read when a number has moved |
| — | **README.md** (this file) | Why it is built this way |

**This file is the design reasoning, not the manual.** If you are looking for how to use the models, you
are in the wrong file — start at [start_here.md](start_here.md).

---

## What this is

PDS is the current policy system (Insurity). It replaced a legacy platform, Velocity, at the start of
2025. Reporting off PDS used to mean a set of report-specific models, each assembling premium in its own
way, which is why two reports could disagree and both look defensible.

This foundation replaces that with **eight fact tables that agree with each other by construction**,
answering three questions the business asks constantly and previously could not answer consistently:

1. **What premium did we write, and what has it earned?** — at any valuation date, on both systems.
2. **What rate did we charge?** — price per unit of whatever the policy is actually rated on.
3. **Did we get rate at renewal?** — is this customer paying more per unit than last year, separating the
   price we charged from the coverage they bought.

Everything else in this folder is detail underneath those three.

---

## The four ideas

Four decisions shape every model here. Each one is easier to see as an example than as a principle, so
each starts with the example.

### 1. A rate is annual premium divided by exposure

A policy insuring 200 apartment units for $50,000 a year is priced at $250 per unit per year. That is the
rate, and $250 is the number an underwriter will argue about.

Generalised:

```
rate = annual premium ÷ (exposure ÷ divisor)
```

**Exposure** is the thing premium is charged per — apartment units, receipts, beds, vehicles. The
**divisor** is how the program quotes it: receipts per $1,000, units per 1.

The word *annual* is load-bearing. **Written premium is not annual premium.** Written premium is what a
transaction booked for the part of the term still to run, so a mid-term endorsement books a fraction of a
year and a cancellation books a negative. Divide a rate into written premium and every endorsement and
cancellation reads as a rate change that never happened.

The consequence runs the other way too: because annual premium is a *rating* construct, it deliberately
does not tie to the ledger. **Anything that must tie to the financials uses written or earned premium,
never the rate numerator.**

### 2. Rate depends on how a policy is rated, not on what line it is

Two General Liability policies in the same program can be priced by completely different mechanisms. A
composite-rated contractor is priced on one negotiated rate over total receipts, and the class-code rates
sitting on the same policy are ignored by the rating engine entirely. Read them anyway and you can be
wrong by a factor of four — on one policy the ignored class-code premium is roughly $16M against a real
premium under $1M.

So the models resolve the **rating method** first, from what the policy actually contains — its premium
types, its coverage types, whether an exposure is present, whether it owns vehicles — and only then work
out what its exposure is. A program nobody has looked at yet either resolves to a known method and is
correct on day one, or publishes no rate and says why.

That guarantee covers the **charged rate**. It does not extend to the ladder: a new program can resolve
correctly, publish a charged rate, and still withhold its normalized rate because a gate it has never been
measured against declines it. The rung then carries its own stated reason rather than nothing — see
[publishing_a_rate.md](publishing_a_rate.md).

**How far "no per-program configuration" actually goes.** The rating-method and exposure-strategy
resolution in `int_pds_policy_rating_method` genuinely contains no subsidiary or program predicate. Three
things around it do, and each is named and tested rather than incidental:

- one program-scoped data-quality correction, in the `is_back_solved_class_code` macro, for TPM
  Contractors policies where an underwriter back-solved a composite price through the class-code rows;
- the composite rate divisor, resolved per program in the fact layer;
- the comparison-group rule, which scopes Count, Flat and Relativity rates by program on purpose — see
  idea 4.

The distinction worth holding onto: **the mechanism is program-agnostic, the units and one correction are
not.**

### 3. Every absent number states its reason

A flat-charge policy has no unit to price. An excess policy carries no exposure of its own anywhere in the
source. An auto cancellation deletes its own vehicle schedule. None of these can have a rate, and none of
them is a defect.

These rows are **kept**, with a text column explaining the absence — `rate_absence_reason`,
`normalized_rate_absence_reason`, `manual_rate_absence_reason`, `bridge_absence_reason` and others. Two
rules follow, and they matter more than any single column:

- **Do not filter out rows with no rate.** They carry real premium — a material share of written premium sits
  on the terms with no rate. Dropping them removes money from every total and sends readers
  looking for source data that never existed.
- **A missing number is never substituted with a plausible one.** A program that records no underwriter
  judgement does not publish "judgement = none" — that would read as a measured finding of zero deviation
  rather than an absence of evidence.

### 4. Rates are only comparable inside a stated group

A contractor's dump truck is not a sports club's van. Both are one unit of `Vehicle Count`, both carry a
rate, and averaging the two produces a number describing nothing.

Rather than expect report authors to know that, every rate carries a **`rate_comparison_group`**, and the
rule is one sentence: **only compare or average rates within the same `rate_comparison_group`.** The
grouping logic lives in the `rate_comparison_group` macro and is asserted by
`assert_rate_comparison_group_is_coherent`.

Dollar and area bases are the permissive case — $1,000 of receipts is $1,000 of receipts in any program —
so those groups do not include the program and cross-program comparison is legitimate. Counts, flat
charges and relativities do include it, because their unit is a business object that differs by program.

**But the permissive case is about the basis type, not the basis name.** Two dollar-denominated bases with
*different names* still land in different groups. A Contractors term rated on `Payroll` and one rated on
`Composite Receipts` are both currency per $1,000 and still do not compare, because payroll and receipts
are different quantities. That is why the group is keyed on the basis and not on the basis type — and it is
also, usefully, what keeps class-code-rated Contractors policies out of the composite-rated comparison,
since in PDS the two rating methods are distinguished by exactly that basis.

One caveat the group deliberately does not cover: it makes the *unit* comparable, not the *risk*. Within
one program and basis, rates still vary many-fold by state and account size. Any headline rate change off
a book average will be mix, and will be believed.

---

## How the models fit together

```
        ┌─ premium & earning ──────────────────────────────────────┐
        │  fact_pds_policy_transaction_premium   (per transaction)     │
        │        └── fact_pds_premium_earned_monthly (× month-end)     │
        └──────────────────────────┬───────────────────────────────┘
                                   │
        ┌─ exposure & rate ────────▼───────────────────────────────┐
        │  fact_pds_policy_exposure_detail   (per basis)               │
        │        └── fact_pds_policy_exposure_rate  (per transaction)  │
        │  fact_pds_policy_rate_component    (per rated row)           │
        │  fact_pds_composite_group_rate     (per negotiated group)    │
        └──────────────────────────┬───────────────────────────────┘
                                   │
        ┌─ the reporting surface ──▼───────────────────────────────┐
        │  fact_pds_policy_term              (per policy term)         │
        │        └── fact_pds_policy_renewal_rate_change (per renewal) │
        │  dim_insured                   (per conformed customer)  │
        └──────────────────────────────────────────────────────────┘
```

Most reports want **`fact_pds_policy_term`**: it is a policy listing with premium, exposure and rate already
attached, and needs no joins. Drop to the facts above it when you need the detail — which coverage moved
the rate, or a valuation date that is not today.

These are **one lineage published at several grains, not a star schema.** The same premium appears in
several of them at different shapes, so joining two facts usually multiplies money rather than adding
columns. [fact_models.md](fact_models.md#join-reference) gives a verdict per join.

---

## Scope

**Covered.** Six programs / lines, plus the legacy platform for premium and rate but not rating mechanics.
See [programs.md](programs.md) for what is complete and what is partial per program.

**Not in production yet.** These models currently build only into development target schemas.
`production.analytics` still carries the older `base_rpt_*` and `rpt_*` reports and nothing here.

**Additive, not a replacement.** Nothing existing was retired. The `base_rpt_*` and `rpt_*` models are
untouched and still serve their consumers, including a production Streamlit app. Migrating those
consumers is separate work. `warn_fact_pds_policy_term_reconciles_to_base_rpt_policy` runs the old and new
side by side and requires every difference to have a named cause, so the two can be compared with
confidence before anything is switched over.

**Guarded by tests, not by convention.** Roughly seventy standalone assertions protect these models, on
top of the grain and column tests declared on every fact — premium identities, the rate contract, and
per-program worked examples pinned to policies traceable in the source. Several encode rules that are
otherwise written down nowhere, so a failing test is usually a real finding rather than a stale
expectation. [fact_models.md](fact_models.md) names the ones that matter per model, and
[extending.md](extending.md) explains how to read a failure.
