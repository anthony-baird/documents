# Glossary

The insurance, pricing and modelling vocabulary used across this folder. Insurance terms first, then the
ones this project invented, then the modelling ones.

---

## Insurance words

**Policy.** One insurance contract. Identified by a policy number.

**Term.** One policy's period of cover, usually twelve months. In this source a policy *is* exactly one
term — a renewal is a new policy number (or the same number with a new suffix, e.g. `-00` then `-01`), not
a second term on the old one. So "policy" and "term" mean the same thing here, and the models call it a
term to be unambiguous.

**Transaction.** Any event that changes a policy. Every policy is a stack of these, and **premium is
booked per transaction, not per policy**:

| Transaction type | What happened |
|---|---|
| **Issue** | The policy was written. The first and usually largest transaction. |
| **Endorsement** | The policy was changed mid-term — more limit, another vehicle, a new location. Books the *extra* premium for the part of the term still to run. |
| **Cancellation** | Cover stopped early. Books a **negative** premium, returning the unused part. |
| **Audit** | After the term, actual exposure is measured against what was estimated, and premium is trued up. Can land a year or more late. |
| **Proposal / Quote** | Not a real policy — a priced offer. **Excluded everywhere**, because these duplicate the Issue premium and would double every total. |

**Endorsement number.** The sequence of a transaction within its policy. `0` is the issuing transaction.

**Named insured.** The customer's name as typed on the policy. **Never join on this.** It is free text
holding a policy's entire schedule of affiliates — frequently dozens of entities and hundreds of
characters — re-entered with different spelling on each transaction. Exact matching fails silently on
exactly the biggest accounts. Use `conformed_insured_key` instead.

**Broker / producer.** The intermediary who placed the business. **Underwriter** is our own employee who
priced and accepted it.

**Line of business (LOB).** The kind of cover: General Liability, Business Auto, Umbrella, Healthcare
Professional Liability.

**Program.** A book of similar business, usually one line for one industry, run by one subsidiary — e.g.
TPM's Contractors GL, KBSI's Senior Living HPL. Pricing mechanics are a property of the *program*, which
is why [programs.md](programs.md) exists.

**Subsidiary.** RSI, TPM or KBSI.

**Limit.** The most the policy will pay. **Deductible** / **SIR** / **retention** — what the insured pays
before cover responds. **Excess** and **umbrella** policies sit *above* another policy's limit and pay
only after it is exhausted.

**Class code.** A standard code for a type of business activity, each carrying its own price. A single
policy usually has several.

**Bureau.** The industry body that publishes expected claim costs per class code per state. Their number
is the market's view of risk, not ours.

---

## The three premiums

Three distinct measures, not three labels for one amount. Full treatment, including why annual premium
cannot be reconciled to the ledger, is in
[start_here.md](start_here.md#3-three-things-these-models-decide-for-you).

| | **Written** | **Annual** | **Earned** |
|---|---|---|---|
| **What it is** | The money a transaction actually booked | What the policy costs for a full year | Written premium recognised across the period it covers |
| **Negative possible?** | Yes — cancellations | No | No |
| **Ties to the accounts?** | **Yes** | **No, deliberately** | Yes |
| **Use it for** | Anything financial | The rate calculation only | Loss ratios, monthly financials |
| **Columns** | `written_premium_*`, `incremental_premium`, `policy_premium` | `rated_premium`, `policy_premium_annual`, `underlying_premium` | `earned_premium`, `unearned_premium` |

**Incremental vs cumulative.** `incremental_premium` is what one transaction booked. Several source-derived
columns are instead running totals for the policy to date — `source_cumulative_written_premium`,
`earned_premium_to_date`.

---

## Exposure and rate

**Exposure.** The quantity of insured stuff that premium is charged per. Apartment units, dollars of
receipts, occupied beds, vehicles, acres, admissions. **Which quantity it is varies by program**, which is
why every row states its own.

**Exposure basis.** The name of that quantity — `Units`, `Composite Receipts`, `Skilled Bed Equivalent`,
`Vehicle Count`. There are 26 of them mapped today.

**Basis type.** Which of five kinds a basis is, and this is what governs comparability:

| Basis type | Meaning | Comparable across programs? |
|---|---|---|
| **Currency** | A dollar amount of exposure | Yes — a dollar is a dollar |
| **Area** | Square footage | Yes |
| **Count** | A count of business objects | **No** — the object differs by program |
| **Flat** | A fixed charge, no unit at all | No rate exists |
| **Relativity** | A percentage of another premium | **No** — different products |

**Divisor.** How the program quotes the price. Receipts are priced per $1,000, units per 1. So the divisor
is what turns a quantity into a quotable unit.

**Rate.** The price of one unit of exposure for one year:

```
rate = annual premium ÷ (exposure ÷ divisor)
```

**Rate unit.** That price expressed in words — "per apartment unit", "per $1,000 of receipts" — so a rate
is readable without looking anything up.

**Comparison group.** A label grouping rates that can legitimately be compared. Two rates in different
groups are different measures with the same name. **Only compare or average rates inside one group.**
Currency and Area group on basis and divisor; Count, Flat and Relativity add the program, because their
unit is a business object that means something different in each.

**Relativity.** Not a price per unit at all, but a percentage. Excess and umbrella policies have no
exposure of their own, so they are priced as a percentage of the underlying primary policy's premium.

---

## The price chain, and the ladder

These describe *how* a rate was built, and the words are easy to conflate.

| Term | What it is | Whose decision |
|---|---|---|
| **Bureau loss cost** | Expected *claim cost* per unit for a class code in a state. No expenses, no profit. | The market's |
| **Loss cost multiplier (LCM)** | What we multiply the loss cost by to make it chargeable — expenses, commission, profit. | **Ours** |
| **Base rate** | Loss cost × LCM. The price per unit before anything about this customer. | Ours |
| **Terms factor** | The effect of what the customer bought. Higher limits load the price up; a deductible credits it down. | The customer's |
| **Judgement** (schedule mod, rate modification) | The underwriter's adjustment for this specific risk. | **The underwriter's** |
| **Charged rate** | Everything above, as actually billed. | — |

"The rate went up" means completely different things depending on which of these moved. The bureau raising
loss costs is the market repricing risk. Us raising the LCM is a decision a person made.

### The ladder — three rates per policy

Each rung answers a different question. Each is obtained by **dividing the charged premium by something**,
not by rebuilding the price upward from the loss cost.

| Rung | The question it answers | How it is derived |
|---|---|---|
| **Charged rate** | What did we actually bill? | Published directly |
| **Manual rate** | What would it have been before the underwriter's judgement? | charged ÷ judgement factor |
| **Normalized rate** | What would it be if every customer bought identical limits and deductible? | charged ÷ terms factor |

**The two derived rungs are independent adjustments off the charged rate, not a chain.** The normalized
rate still contains the underwriter's judgement, deliberately — it is the rate actually charged, restated
to standard terms. Its purpose is to separate a price movement from a coverage movement: a renewal at last
year's rate on 6% more limit is a ~6% price reduction that the charged rate reports as flat.

**Normalization basis.** Standard terms sit at different ends of the factor range on different programs,
so normalizing pushes rates *down* on the GL and auto programs and *up* on senior living. That difference
is pure arithmetic and invisible in the number itself. **Never compare normalized rates across
normalization bases.**

---

## Words this project invented

**Rating method.** How a policy is actually priced, resolved from what the policy contains rather than
from which program it belongs to: `class_code`, `composite`, `multiple_composite`, `bed_equivalent`,
`vehicle_count`, `hired_non_owned_only`, `underlying_relativity`, `flat_charge`, `negotiated_premium`,
`no_exposure_available`.

**Composite rate.** One negotiated rate applied to a whole account's receipts, replacing the class-code
rates entirely. The class-code rows still sit in the source and **are not premium** — the rating engine
ignores them.

**Build-up rows.** Underwriting's working, where they priced every class code before collapsing it to one
negotiated composite rate. Live in the source, excluded from the policy total, and vastly larger than real
premium. Published for audit only.

**Absence reason.** A text column explaining why a number is missing. There are eight of them across the rate
models — `rate_absence_reason`, `normalized_rate_absence_reason`, `manual_rate_absence_reason`,
`terms_factor_absence_reason`, `judgement_factor_absence_reason`, `row_achieved_rate_absence_reason`,
`schedule_mod_absence_reason`, `rate_change_absence_reason` — plus `bridge_absence_reason` on `dim_insured`.
Every string they can carry is catalogued in [publishing_a_rate.md](publishing_a_rate.md). The rule behind
them: **a missing number is never replaced with a plausible one.** A program that records no underwriter
judgement publishes an absence, not "judgement = none", because the latter reads as a measured finding of
zero.

**Reportable.** A gate, not a caveat — and there are **two** gates, testing different things. Check the right
one.

- `rate_is_reportable` asks whether *this policy's* rate is a price at all: is the exposure positive, is the
  premium positive, and does the basis it was struck on represent enough of the policy. It says nothing about
  comparability, and nothing about whether the ladder is available.
- `rate_change_is_reportable` asks whether *two* rates can be divided — whether both are real prices and both
  are measured in the same unit. A payroll-rated term against a receipts-rated one fails here, not above.

Both are stated in full, with their thresholds, in [publishing_a_rate.md](publishing_a_rate.md).

**Rate as of.** The term carries one rate, and it *moves* — an audit or endorsement booked later changes
what the policy cost per unit of exposure. `rate_as_of` is the date it was valued at, published beside it
rather than left implicit, so a stale extract identifies itself instead of appearing current.
`development_age_months` says how mature the term was at that date.

**Conformed insured key.** One customer identity spanning both policy systems, built from the predecessor
policy number the source itself records. No fuzzy matching, no name matching, no tax-id matching anywhere
in it.

**Cutover.** The start of 2025, when PDS replaced Velocity. Comparisons spanning it are the hardest thing
in this domain, because the two systems record different things.

---

## Modelling words

**Grain.** What one row of a table represents — "one row per policy term", "one row per transaction ×
month-end". Determines whether a column can be summed without double-counting, so every model section in
[fact_models.md](fact_models.md) states it first.

**Fan-out.** A join multiplying rows because the right-hand side has several matches per left-hand row,
counting money more than once. Detectable in one query by comparing row counts either side of the join;
not detectable by eye in the output.

**Memo column.** Published for information, deliberately **not additive**. `build_up_premium`,
`overridden_class_code_premium`, `liquor_premium`, `underlying_auto_premium`.

**Derived vs sourced.** A sourced number was read from the policy system. A derived one was computed here
because the source does not record it. Derived numbers are labelled, and must never be presented as source
values.

**Closed form.** A formula evaluated directly instead of storing a row per period. The earning models store
parameters — a start date, a day count, a daily amount — so earned premium at any date is
`days elapsed × daily amount`, with no row-per-month spine required.

**Premium-weighted harmonic mean.** How the factor columns are aggregated. The operative consequence is
that they must not be averaged again downstream — an average of an average of ratios is a different
quantity.

**Staging / intermediate / fact.** The layers. `stg_pds_*` is a light rename of one source table.
`int_*` holds business logic. `fact_*` and `dim_*` are what reports read. Full conventions are in
`CLAUDE.md`.

**Assertion test.** A SQL file under `tests/` that returns rows only when something is wrong. Several here
encode rules written down nowhere else, so a failing test is usually a real finding rather than a stale
expectation.
