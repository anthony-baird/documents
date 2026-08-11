# What these models were built to answer

Ten worked examples, from the most common to the most specialised. Each one names the model it reads and
its grain, the shape of the query, and the mistake that would make the answer wrong.

[start_here.md](start_here.md) covers where these tables live and the three decisions the models encode.
Vocabulary is in [glossary.md](glossary.md).

The SQL is illustrative. Column names are real; table names are unqualified, so prefix them with your own
target schema. **Grain matters more than anything else on this page** — it is what decides whether you can
sum a column without double-counting, so each example states it.

---

## 1. Monthly earned premium by program

**Reads.** `fact_pds_premium_earned_monthly` — one row per transaction × month-end, within that transaction's
own earning window.

**The question.** What earned premium do we recognise in each month, by subsidiary and program?

```sql
select    subsidiary, program, month_end_date,
          sum(earned_premium_in_month) as earned_premium
from      fact_pds_premium_earned_monthly
group by  1, 2, 3
```

**The mistake.** Summing a `_to_date` column alongside it. Both `earned_premium_to_date` and
`written_premium_to_date` are stated **per row, as at that month-end** — the same transaction's figure is
repeated on every month-end in its earning window, ten times over on average. Summing either across months
compounds: `sum(written_premium_to_date)` over the whole fact returns about $30.7bn against a true written
premium of about $2.46bn. Take one month's rows for a balance; take `earned_premium_in_month` for a
movement. There is no written-premium movement column here, because a transaction is written once and this
fact has no row that is unique to the month it was written in.

Second mistake: if you left-join a full calendar, expect flat lines rather than zeros after a transaction's
window closes — absence means "no further earning", not "nothing earned".

**Written premium by month comes from elsewhere.** Written premium belongs to the month a transaction was
*booked*, which is a property of the transaction rather than of a month-end, so read the transaction grain:

```sql
select    subsidiary, program,
          last_day(transaction_booked_date) as month_end_date,
          sum(incremental_premium)          as written_premium_in_month
from      fact_pds_policy_transaction_premium
group by  1, 2, 3
```

Do not try to recover this from the monthly fact by filtering it to each transaction's booked month-end.
A transaction can be booked in a month that falls outside its own earning window — an audit booked after
expiry, a back-dated endorsement — so that month-end row does not exist and the premium silently
disappears. It loses about $13.9M today. The query above reconciles to the transaction fact exactly.

---

## 2. Earned and unearned premium at an arbitrary valuation date

**Reads.** `fact_pds_policy_transaction_premium` — one row per policy transaction, both systems.

**The question.** What was earned and unearned as at the 12th of last month?

The monthly fact cannot answer this, because it only carries month-ends. Instead this model publishes the
*parameters* of earning — a start date, a day count and a daily amount — so earned premium at any date is
just `days elapsed × daily amount`. Compute it directly:

```sql
select    sum(incremental_premium_fully_earned
              * iff(transaction_effective_date <= :as_of, 1, 0))
        + sum(daily_earn_amount
              * greatest(0, least(datediff('day', earn_start_date, :as_of) + 1, earn_days)))
            as earned_premium,
          sum(iff(transaction_booked_date <= :as_of, incremental_premium, 0))
            as written_premium
from      fact_pds_policy_transaction_premium
where     earn_start_date <= :as_of
```

**The mistake.** Forgetting the fully-earned component, which is recognised whole at its effective date
rather than spread. On some programs it is a material share of premium.

Second mistake: dropping the `+ 1` on the `datediff`. The window is inclusive of **both** endpoints — a
transaction earns on its start date — so `datediff` alone is one day short on every open window. It
understates earned premium by around $1.6M across the book, and puts the figure out of step with
`fact_pds_premium_earned_monthly`, which `assert_monthly_earned_matches_closed_form` holds to the
inclusive count.

Third mistake: writing `written_premium` as a bare `sum(incremental_premium)` and letting the `where`
clause date it. That filters written premium on the **earning** axis, which counts premium not yet booked
at the valuation date — about $7M too much at mid-2026. Written premium is dated by
`transaction_booked_date`, as above, and the two axes do not coincide.

---

## 3. A policy listing for the business

**Reads.** `fact_pds_policy_term` — one row per policy term. No joins needed; this is the model most reports
should start from.

**The question.** Every policy written in the current program year, with premium, exposure and price.

```sql
select    policy_number, named_insured, broker_name, underwriter,
          policy_inception_date, policy_expiration_date, is_cancelled,
          written_premium,
          exposure_amount, exposure_basis,
          rate, rate_unit, rate_as_of,
          rate_is_reportable
from      fact_pds_policy_term
where     policy_inception_date >= date_trunc('year', current_date())
```

**`program_year` is not a calendar year.** It is the treaty year *ordinal* for that subsidiary and program
— a small integer, currently 1 to 9 — so `where program_year = 2026` compiles and returns **nothing**.
Filter on `policy_inception_date` for a calendar period, and read `program_year` only as a within-program
sequence number. The ordinal is not comparable across programs: two policies both in program year 4 are
in the same treaty year only if they share a subsidiary and program.

**The mistake.** Adding a `where rate is not null` filter to tidy the output. Those rows carry
real premium — flat-charge policies, excess policies, hired-and-non-owned auto — and dropping them removes
money from the total. A material share of written premium sits on the terms with no rate.

Show `rate_is_reportable` instead, so the unpriced rows stay visible and labelled — and see
[publishing_a_rate.md](publishing_a_rate.md) for what that gate tests and every reason a rate is absent.

Note that this model carries **no charged-rate absence reason** — only
`manual_rate_absence_reason` and `normalized_rate_absence_reason`. To get the
plain-words reason a charged rate is missing you have to go to the transaction grain, and it is available
for PDS terms only:

```sql
select    t.policy_number, t.written_premium, r.rate_absence_reason
from      fact_pds_policy_term t
left join fact_pds_policy_exposure_rate r
       on  r.policy_key = t.policy_key
where     t.rate is null
```

That join is at policy grain, so a policy multiplies by its transaction count — aggregate or pick a
transaction if you need one row per term. Legacy terms return no reason at all, because the rate fact holds
PDS only.

---

## 4. Rate benchmarking — price per door, per bed, per vehicle

**Reads.** `fact_pds_policy_term` — one row per policy term.

**The question.** What is the spread of rates we are charging, and which accounts sit at the extremes?

```sql
select    rate_comparison_group,
          count(*)                                        as policies,
          median(rate)                                    as median_rate,
          percentile_cont(0.10) within group (order by rate) as p10,
          percentile_cont(0.90) within group (order by rate) as p90
from      fact_pds_policy_term
where     rate_is_reportable
group by  1
```

**The mistake.** Grouping by program or line instead of by the comparison group. A single group is the
only scope in which a median rate means anything — mix apartment units with square footage, or one
program's vehicles with another's, and the median describes nothing.

Even inside one group, remember that the group makes the *unit* comparable and not the *risk*. Rates vary
many-fold by state and account size within a single program.

---

## 5. Did we get rate at renewal?

**Reads.** `fact_pds_policy_renewal_rate_change` — one row per renewal pair (customer × line × current term
that has a predecessor). Covers general liability on Contractors, Habitational and Sports & Entertainment;
Business Auto and Excess on every program; and Healthcare Professional Liability on Senior Living (KBSI).

**The question.** For customers who renewed, are we charging more per unit than last year?

```sql
select    program, conformed_insured_key, current_policy_number,
          prior_rate, current_rate, rate_change_pct,
          prior_normalized_rate, current_normalized_rate, normalized_rate_change_pct,
          terms_changed_at_renewal
from      fact_pds_policy_renewal_rate_change
where     rate_change_is_reportable
```

**The mistake.** Averaging `rate_change_pct` to a headline number. A book average is mostly **mix** — new
business prices lower, so the book median can fall sharply in a period when every renewing account went
up. Report per-pair, or per stated segment with the scope on the face of the report.

Second mistake: reading `rate_change_pct` without the gate. A term rated on payroll compared against one
rated on receipts produces arithmetic rather than a price change.

Third mistake: pooling books. `program` and line are in the select list because each book refuses a
comparison for its own reason (and some, like auto and excess, refuse none), so a mixed result set holds
several different populations. Note also that a normalized rate change is only ever populated for general
liability and for Senior Living Healthcare Professional Liability — and the two must not be pooled, because
senior living normalizes *up* and general liability normalizes *down*.

---

## 6. Separating a price increase from a coverage change

**Reads.** `fact_pds_policy_renewal_rate_change` — one row per renewal pair.

**The question.** This account's rate is flat year on year. Did we actually hold price?

This is the case the ladder exists for. The raw rate answers "what did we bill per unit"; the normalized
rate answers "what would we have billed if they had bought the same coverage as last year". Compare them:

```sql
select    current_policy_number,
          rate_change_pct,               -- what the raw rate did
          terms_factor_change_pct,       -- what the customer's limits/deductible did
          normalized_rate_change_pct     -- what the price actually did
from      fact_pds_policy_renewal_rate_change
where     rate_change_is_reportable
  and     normalization_basis_comparable
order by  abs(normalized_rate_change_pct - rate_change_pct) desc
```

The top of that list is the supervision report. A pair can show a 0% rate change and a ~6% price *cut*,
because the insured bought 6% more limit at the same rate. Another can read +5% raw and close to +20%
true.

**The mistake.** Expecting this on every pair. `normalized_rate_change_pct` needs a recorded factor stack
on **both** sides, and pre-cutover terms have none. Most pairs are unavailable today, and that is a
population limit rather than a defect — the coverage grows as the current system accumulates renewals. On
legacy pairs, `terms_changed_at_renewal` still warns that terms moved, without fabricating a magnitude.

**The other mistake: reading one book's rules into another.** These examples span several books — general
liability on Contractors, Habitational and Sports & Entertainment (which refuse a comparison for rating
method, almost nothing, and a change of exposure unit respectively), plus Business Auto, Excess and KBSI
Healthcare Professional Liability (which refuse nothing on comparability). Filter by `program` and line
unless you mean the whole set, and on Sports & Entertainment decide whether you want the pairs comparable
only on our judgement that a unit was renamed (`basis_comparable_by_renamed_unit`). See
[fact_models.md](fact_models.md) and [programs.md](programs.md).

---

## 7. Why did the rate move on this account?

**Reads.** `fact_pds_policy_rate_component` — one row per rated premium row: a coverage on the class-code
programs, a location on senior living, a vehicle-coverage line on auto.

**The question.** This policy's rate rose 12%. Which coverage, and through which factor?

The policy-level facts tell you *whether* price moved. Only this grain tells you what moved it:

```sql
select    coverage_type, rate_line, premium_amount_type,
          annual_premium, row_exposure_amount,
          bureau_loss_cost, loss_cost_multiplier, base_rate,
          increased_limit_factor, deductible_credit,
          rate_modification_factor, row_achieved_rate
from      fact_pds_policy_rate_component
where     policy_transaction_id = :transaction
order by  annual_premium desc
```

Reading down the chain separates the three causes that get conflated: the bureau raising loss costs (the
market), us raising the loss cost multiplier (our decision), and the customer buying different limits.

**The mistake.** Interpreting the factor columns without reading `rate_line` first. On senior living the
deductible column is a multiplier and the limit column reduces the price — the opposite of GL on both
counts.

---

## 8. Composite-rated contractors

**Reads.** `fact_pds_composite_group_rate` — one row per transaction × composite group. Contractors only, by
construction.

**The question.** What rate did each negotiated group get on this multi-group policy?

```sql
select    g.composite_group_number, g.composite_group_name,
          g.charged_premium_annual, g.exposure_amount,
          g.rate, g.terms_factor, g.normalized_rate, g.rate_absence_reason
from      fact_pds_composite_group_rate g
where     g.policy_transaction_id = :transaction
```

**The mistake.** Trying to get this from `fact_pds_policy_exposure_rate`, which deliberately withholds a rate
and a terms factor on these policies and names this model in the absence reason. A policy-level rate across
several separately negotiated groups is not a price of anything.

Note this model has gates of its own — a group rate outside a plausible band is withheld, as is its normalized
rate where too little of the build-up premium carries a limit factor. Both reasons, and their numbers, are in
[publishing_a_rate.md](publishing_a_rate.md).

Second mistake: comparing `build_up_premium` against `charged_premium_annual` and concluding premium is
missing. The build-up is the rating engine's working, includes templated duplicate rows, and is published
only so the terms factor's weighting can be audited.

---

## 9. Excess and umbrella pricing

**Reads.** `fact_pds_policy_exposure_rate` — one row per policy transaction, PDS only.

**The question.** What percentage of the underlying primary premium are we charging, by program?

```sql
select    program,
          count(*)                     as transactions,
          median(pct_of_underlying)    as median_relativity
from      fact_pds_policy_exposure_rate
where     basis_type = 'Relativity'
  and     rate_is_reportable
group by  1
```

Median relativities sit in a fairly narrow band — roughly 29% on senior living, mid-30s on sports and
entertainment and habitational, and around 45% on contractors. Four programs write reportable relativities
today, not all six.

**The mistake.** Reading the spread as several estimates of one number and picking an average. They are
different products with different attachment structures, which is why `rate_comparison_group` scopes
relativity by program. A movement across the cutover on one tower is the platform change itself, not a
price change.

Second mistake: taking a `max()` or a mean rather than a median. Individual relativities reach several
multiples of the underlying premium on thin towers and are still flagged reportable, so any statistic
sensitive to the tail describes those few rows rather than the book.

Third mistake: including auto premium in the denominator. It is carried as
`underlying_auto_premium` for information, and folding it in would move the relativity only on the
accounts that happen to place auto with us.

---

## 10. Mixed-basis policies

**Reads.** `fact_pds_policy_exposure_detail` — one row per transaction × exposure basis × divisor. This is the
longest-grained exposure model and the only place a mixed-basis policy's rate is readable per basis.

**The question.** This policy shows no rate, or shows a rate on only one of its exposures. What is it
actually rated on?

```sql
select    exposure_basis, basis_type, rate_divisor,
          exposure_amount, rated_premium, rate, rate_unit,
          rate_comparison_group
from      fact_pds_policy_exposure_detail
where     policy_transaction_id = :transaction
order by  rated_premium desc
```

The policy-level fact publishes one rate. Where several units of genuinely different kinds are rated
together and one carries at least three quarters of the premium, it publishes that one — never a blend
across kinds. Where nothing dominates, it publishes nothing and points here.

**The mistake.** Summing `exposure_amount` across the rows to get "total exposure". Square feet plus
apartment units is not a quantity.

---

## Where to start

| If you want | Read |
|---|---|
| A policy listing, premium, exposure and rate together | `fact_pds_policy_term` |
| A monthly financial figure | `fact_pds_premium_earned_monthly` |
| A valuation date that is not a month-end | `fact_pds_policy_transaction_premium` |
| Rate benchmarking or a rate distribution | `fact_pds_policy_term`, or `fact_pds_policy_exposure_rate` for transaction detail |
| Rate change at renewal | `fact_pds_policy_renewal_rate_change` |
| Why a rate moved | `fact_pds_policy_rate_component` |
| A mixed-basis or unrated policy explained | `fact_pds_policy_exposure_detail` |
| A composite contractor's group rates | `fact_pds_composite_group_rate` |
| Customer identity across the two systems | `dim_insured` |
