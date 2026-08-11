# Start here

The PDS foundation publishes eight fact models covering premium, exposure and price. This page covers
where they are, which one to query, and the three decisions they encode that a report has to work with.

[glossary.md](glossary.md) carries the insurance and modelling vocabulary used across this folder.

## 1. Find the tables

These models are built by dbt into **your own target schema**. There is no suffix on them — a fact or
dimension lands in the bare target schema, while only the `5_reporting` models get a subsidiary suffix.

To see where yours are:

```sql
select current_database(), current_schema();
show tables like 'FACT_PDS_%';
```

These build into a personal development schema — `development.dbt_abaird` for the author of this project.

**These models are not in production yet.** `production.analytics` still carries only the older
`base_rpt_*` and `rpt_*` reports. Everything on this page is a development build, so do not point a
business dashboard at it without agreeing that first. See
[fact_models.md](fact_models.md#relationship-to-existing-models) for what the old models still own.

## 2. Your first query

`fact_pds_policy_term` is one row per policy term, with premium, exposure and price already attached. It needs
no joins. Most reports should start here.

```sql
select    policy_number,
          named_insured,
          subsidiary,
          program,
          policy_inception_date,
          policy_expiration_date,
          written_premium,
          exposure_amount,
          exposure_basis,
          rate,
          rate_unit,
          rate_as_of
from      fact_pds_policy_term
where     policy_inception_date >= date_trunc('year', current_date())
order by  written_premium desc
limit     50;
```

That is a policy listing, and it needs no joins because the term fact resolves the exposure and rate itself.

Note what the filter is **not**. The model has a `program_year` column, but it holds the treaty year
*ordinal* for that subsidiary and program — a small integer, currently 1 to 9 — so `where program_year =
2026` compiles and returns nothing at all. Use `policy_inception_date` for any calendar period.

One thing to notice in the output: `exposure_basis` **changes from row to row** — `Units`,
`Composite Receipts`, `Skilled Bed Equivalent`, `Vehicle Count`. Exposure is whatever the policy is rated
on, so the basis column is what makes `exposure_amount` and `rate` interpretable, and
`rate_unit` states the rate's denominator in words. Which basis a policy gets is resolved from
its contents, not configured per program — see [programs.md](programs.md).

## 3. Three things these models decide for you

Each of these is a choice made in the models rather than left to the report. Working against one of them
is where most wrong numbers come from.

### 1 — The rate is one measure, valued at a stated date

Every model here carries more than one kind of premium simultaneously, and they are not interchangeable
labels for one amount:

| Naming | Measure | Definition |
|---|---|---|
| `written_*`, `incremental_premium`, `policy_premium` | Written | What a transaction booked for the part of the term still to run. Negative on cancellations. |
| `*_annual`, `rated_premium`, `underlying_premium` | Annual | What the policy costs for a full year. |
| `earned_*`, `unearned_premium` | Earned | Written premium recognised across the period it covers. |

Two consequences that are properties of this design rather than of premium generally:

- **Annual premium is a rating construct and does not tie to the ledger — deliberately.** It exists because
  a mid-term endorsement books a fraction of a year and a cancellation books a negative, neither of which
  can be divided into a price per year. So `rate × exposure` will not reconcile to written premium, and
  chasing that gap is wasted effort. Anything financial uses written or earned.
- **The term carries ONE rate, and it is valuation-dated.** `rate` is the price per unit of exposure implied
  by everything booked on the term so far, including any audit that has landed, and `rate_as_of` says when it
  was valued. It moves as audits and endorsements land, which is why the as-of travels with it.
  `written_premium` and `written_premium_all_transactions` overlap rather than partition — they differ only
  where premium was booked after the as-of date. `incremental_premium_fully_earned` and
  `incremental_premium_pro_rata` are likewise components of `incremental_premium`, not additions to it.

### 2 — Rows with no rate are retained, and the reason is published somewhere else

`rate` is NULL on a few hundred of the terms in the book, carrying a material share of written premium. Those
rows are kept rather than excluded, because the absence is a property of the product: flat-charge policies
have no unit to price, excess and umbrella policies carry no exposure of their own anywhere in the source,
and an auto cancellation deletes its own vehicle schedule.

The part that cannot be inferred is **where the explanation lives**. `fact_pds_policy_term` publishes absence
reasons for the manual and normalized rates but **none for the charged rate**, so it can tell you a rate is
unreportable and not why. That sits at transaction grain:

```sql
select    t.policy_number, t.written_premium, r.rate_absence_reason
from      fact_pds_policy_term t
left join fact_pds_policy_exposure_rate r
       on  r.policy_key = t.policy_key
where     t.rate is null
```

Around **half of them are pre-2025 legacy terms and can never carry a reason**, because
`fact_pds_policy_exposure_rate` holds PDS transactions only. The join is at policy grain rather than to one
chosen transaction, so it can return several rows per term.

Use `rate_is_reportable` as the gate for rate work, and no rate filter at all for premium
totals. What that gate actually tests, and every reason a rate or a ladder rung can be absent, is in
[publishing_a_rate.md](publishing_a_rate.md).

### 3 — Only compare rates inside the same comparison group

A rate is a price per unit, so two rates are only comparable if the unit means the same thing. $1,000 of
receipts is $1,000 of receipts anywhere. A *unit* is not: a contractor's dump truck is not a sports
club's van, and an apartment unit is not whatever a sports venue counts.

That comparability is resolved in the models rather than left to the report. Every rate carries
**`rate_comparison_group`** on the term fact and on the
transaction-grain facts, and the rule is one sentence:

> **Only compare or average rates within one comparison group.**

```sql
select    rate_comparison_group,
          count(*)      as terms,
          median(rate)  as median_rate
from      fact_pds_policy_term
where     rate_is_reportable
group by  1
order by  terms desc
```

One thing the group does **not** do: it makes the *unit* comparable, not the *risk*. Inside a single group
rates still vary many times over by state and account size. Any headline rate change taken off a book
average is mostly a change in business mix, and it will be believed.

## 4. Where to go next

| If you want | Read |
|---|---|
| A word defined | [glossary.md](glossary.md) |
| Ten worked queries with the mistake spelled out for each | [analytics_examples.md](analytics_examples.md) |
| What each fact model contains and how they relate | [fact_models.md](fact_models.md) |
| How your program is actually priced, and its one trap | [programs.md](programs.md) |
| To write SQL against the raw `stg_pds_*` tables | [relational_model.md](relational_model.md) |
| To add a program, a column, or a new exposure unit | [extending.md](extending.md) |
| The design reasoning behind all of it | [README.md](README.md) |

## The shape of the model set

These eight facts are **one lineage published at several grains, not a star schema.** The same premium
appears in several of them at different shapes, which is why most pairs are not meant to be joined and
[fact_models.md](fact_models.md#join-reference) gives a verdict per pair rather than a diagram.

The join in decision 2 above shows what makes one of them safe. `fact_pds_policy_term` is one row per policy
term and `fact_pds_policy_exposure_rate` is one row per transaction, so it is the
`and r.transaction_type = 'Issue'` predicate — not the join key — that holds the result at one row per term.

So: before you join any two of these models, check the verdict table in
[fact_models.md](fact_models.md#join-reference), and count your rows before and after.
