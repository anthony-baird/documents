# The programs, and how each one is priced

Six programs / lines of business, across three subsidiaries. They are documented separately because
**they are not variations on one pricing model** — the same column can mean opposite things on two of
them, and a report that treats them uniformly will be wrong on at least one.

**How to use this file.** Find the program, read its four bold lines. Every section is laid out the same
way:

1. **Who writes it**, and what kind of business it is.
2. **Priced on** — the exposure unit, and the mechanism.
3. **The trap** — the one property of the program that most often produces a wrong number.
4. **The ladder** — which of the three rates are available, and which are absent and why.

Vocabulary is in [glossary.md](glossary.md) — particularly *exposure*, *basis*, *the ladder*, *terms factor*
and *normalization*, which this file uses throughout.

| Program / line | Subsidiary | Priced on |
|---|---|---|
| [Contractors GL](#contractors-general-liability) | TPM | Receipts, at one negotiated composite rate |
| [Habitational GL](#habitational-general-liability) | TPM | Apartment units and other class-code exposures |
| [Sports & Entertainment GL](#sports--entertainment-general-liability) | RSI | ~20 different exposure units |
| [Senior Living HPL](#senior-living-healthcare-professional-liability) | KBSI | Weighted bed count |
| [Business Auto](#business-auto) | All three | Vehicles |
| [Umbrella / Excess](#umbrella--excess) | TPM, RSI, KBSI | A percentage of the underlying policy |
| [Legacy Velocity](#legacy-velocity-pre-2025) | — | (a platform, not a program) |

Relative scale, rounded, for a sense of materiality only — these are measured on different premium bases
at different dates and are **not** a set of comparable figures:

```
Contractors GL       ~$720M      the largest book
Business Auto        ~$540M      across all subsidiaries; ~86% of it is Liability
Umbrella / Excess    ~$455M
Senior Living HPL    ~$390M
Habitational GL      ~$360M
Sports & Ent. GL     ~$200M      issuing transactions; endorsements add substantially more
```

---

## Before the programs: one thing that is program-specific

The pricing vocabulary — bureau loss cost, loss cost multiplier, base rate, terms, judgement, charged rate
— and the three-rung **ladder** of charged, manual and normalized rates are defined once in
[glossary.md](glossary.md#the-price-chain-and-the-ladder). Read that first if any of those terms is new;
this file assumes them.

The one part of it you cannot learn from the glossary, because it differs per program, is **which
direction normalization moves a rate.**

**Normalized rates are comparable within a normalization basis and never across one.** On the GL and auto
programs, standard terms sit at the *bottom* of the factor range, so normalizing restates rates **down** —
roughly halving them. On senior living the same factors run the other way and normalizing restates
rates **up**, about 1.6×. That difference is pure arithmetic and is invisible in the number, which is why
every row carries `normalization_basis`.

Put a senior living normalized rate beside a GL one and the senior living book will look expensive by
construction. Nothing in the number itself will warn you.

---

## Contractors General Liability

**TPM.** Construction and contractors liability, and the largest book in the business. Heavily New York
Free Trade Zone — a segment exempt from filed rate and form requirements — and that segment is
overwhelmingly New York risk.

**Priced on receipts, per $1,000, at a single negotiated rate.** Underwriting builds the price up from
every applicable class code, collapses it to one rate over total receipts, then **negotiates that rate**
against the account's loss experience and venue. The negotiated rate applied to receipts is the premium.

**The trap: the class-code build-up rows are not premium.** They are live in the source, and the rating
engine excludes them from the policy total. Summed naively they exceed the entire book. The stored
"composite rate" field is likewise a derivation artefact and not the price charged. Both are published
as memo columns only and are labelled as such.

A minority of these policies look class-code rated in the data and are not: an underwriter back-solved a
composite price by running the build-up at full manual rate and booking a large **negative** manual
adjustment to bring the total down. The models net that plug in rather than excluding the policies.

**The ladder:** charged rate yes; **no manual rate** — the negotiation *is* the judgement, so there is
nothing to divide out; normalized rate derived from the build-up rows.

---

## Habitational General Liability

**TPM.** Apartment and habitational liability. Straightforwardly class-code rated — one layer, no
composite override, no shadow build-up. **The cleanest program in the book, and the reference case for
everything else.**

**Priced on apartment units** primarily, with square footage, discrete features and tenant receipts rated
alongside. Units carry the overwhelming majority of premium, so the published rate is the price per door —
which is what underwriters actually quote.

The full price chain is recorded on every rated row, and the published rate reproduces the rating
engine's own arithmetic across every transaction type including cancellations.

**The trap: the charged rate is nearly silent, because price and terms moved in opposite directions by
similar amounts.** Between the last two program years the filed price fell close to a fifth, while
insureds bought materially more limit. The achieved rate moved only a few percent. Read the charged rate
alone and pricing looks flat; it is not.

**No underwriting judgement is recorded anywhere on this program** — seven separate candidate factors are
all pinned at exactly neutral across the whole book. The load is most likely absorbed into loss cost
selection or the LCM, in which case it is not recoverable retrospectively.

**The ladder:** charged rate yes; **manual rate absent, with a stated reason** (not set equal to the
charged rate); normalized rate exact on every transaction.

**Renewal rate change: the easiest book to read.** Because the program is rated per door throughout, on
both the legacy and the current platform, almost every renewal pair is comparable and very little is
withheld. The only refusals are a handful of terms that move from being rated on units alone to a blend of
units and something else. A large share of the pairs bridge the 2023 platform change, which is what the
cross-system customer bridge was built for — and given the trap above, read the terms-adjusted change rather
than the raw one.

One loose end on those few refusals, recorded so it is not rediscovered as a defect: the recorded unit count
barely moves across them, which is the same signature that identifies a renamed unit on sports &
entertainment. So the blend may be a relabelling rather than a genuine change of rating basis. They are
withheld conservatively; nobody has established which it is, and the renamed-unit allowance is deliberately
not extended here.

---

## Sports & Entertainment General Liability

**RSI.** Sports and entertainment liability, class-code rated, and the most varied book in the business —
around **twenty different exposure units**: gross sales, area, members, attendants, acres, admissions, boats,
camper days, events, and more.

Those units are not noise to be normalized away. They carry genuinely different prices — an attendant is
priced in cents where an event is priced in hundreds of dollars. Where a policy mixes units and one
carries the clear majority of premium, the model publishes that unit's own rate; where nothing dominates,
it publishes **no rate rather than a meaningless blend**.

**Liquor liability** is rated as a separate line inside this program, on its own price chain. It is
correctly a class code within the same rate rather than a standalone product, and there is a test holding
that.

**The trap: nearly a fifth of premium carries no stored rate at all, deliberately.** The rate was
overridden at entry and the source kept only the resulting premium. So the published rate is derived from
premium and exposure rather than read from the chain — which is why it exists on this program at all.

**The ladder:** all three rungs. But the normalized rate on this program is **reconstructed**, not read from
each policy's own rating chain. This is the one place the ladder departs from "read the recorded factor", so
it is worth understanding why.

### Why the normalized rate is not read straight from the rating chain

The normalized rate answers "what would this rate be at the program standard — $1M/$2M limits, first
dollar". The natural way to get it is per policy: every rated row records an increased-limit factor and a
retention credit, so `normalized rate = charged rate ÷ (increased-limit factor − retention credit)` would
restate each policy to basic limits using its own chain. That is exactly how Habitational and the legacy
programs do it, and where the chain is recorded consistently it is the right method.

**It cannot be done that way on RSI, because RSI records the increased-limit factor inconsistently.** The
platform progressively stopped folding the limit factor into the base rate and began recording it as an
explicit factor. So for identical coverage the recorded factor drifts from about **0.9 on older terms to
about 2.0 on recent ones** — a term-by-term difference in *bookkeeping*, not in coverage. The evidence is
unambiguous:

- The recorded factor at fixed $2M/$4M limits runs a median of 0.93 in 2023, 1.50 in 2024, 1.92 in 2025,
  2.04 in 2026 — the limit load migrating out of the base rate and into the explicit factor over time.
- Within a single limit cell the factor is *bimodal*: a cluster near the retention credit (limit factor
  baked away) and a cluster near 2.0 (limit factor explicit), with an empty gap between — two bookkeeping
  conventions, not a spread of real factors.
- Whole states record it baked even recently: New York deductible terms carry a recorded limit factor of
  exactly 1.0 on thousands of rows.

Dividing each policy by its own recorded factor therefore **under-adjusts the baked terms** (factor near 1,
so almost nothing is removed) and **over-adjusts the explicit ones** (factor near 2). At renewal this is
precisely how it mishandles a limit change: a policy whose limits never moved, but which crossed from baked
to explicit between its two terms, reads a ~50% normalized "price cut" that never happened. The charged rate
did not move; only the way the price was split into base × factor did.

**The reconstruction sidesteps this because the charged rate is convention-proof.** The charged rate is
premium ÷ exposure — the same total whichever way the price was split — so it is trustworthy. What is not
trustworthy is the *decomposition*. So instead of each policy's own factor, the model divides the charged
rate by a factor **rebuilt from the limits and retention the policy bought**, taken as a robust median over
recent RSI terms at those limits and retention (recent, because recent terms record the factor explicitly;
median, because it ignores the few stragglers that do not). The factor is expressed relative to a $1M/$2M
first-dollar term in the same state, so a policy already at the standard reads 1.0 and its normalized rate
equals its charged rate. See `int_pds_rsi_reconstructed_terms_factor`.

This is an **estimate** — a fitted central factor for a group, good enough to restate a rate to standard
terms, not to price a policy — and it is honest about its edges: state-specific factors are used only where
a state has enough terms, otherwise a national factor stands in; self-insured-retention terms are **withheld
entirely** (too few, and their credit does not fall monotonically with retention, so no sensible factor can
be fitted); and a limits/retention combination with no recent support publishes no normalized rate, with a
stated reason. About 87% of reportable RSI terms resolve a reconstructed normalized rate; the rest say why
not.

**This reconstruction is interim, and is intended to be replaced.** RSI files ISO's published increased-limit
factor tables — GL-2022-BPOP1 for premises/operations and GL-2022-BPRD1 for products/completed operations —
so the limit factor is a *known* value, not something that needs fitting. The intended build reads those
tables directly: for a policy's limits, look up each subline's factor and blend the two by the policy's own
premises/ops-vs-products premium split. That would be exact rather than estimated, cover essentially every
term rather than 87%, need no state cells or national fallback, and recover the self-insured-retention terms
this version withholds (ISO publishes the deductible and SIR credit tables too). It is not built yet because
it needs the licensed ISO table values loaded as a reference, and a clean premises/ops-vs-products premium
split, which the current source models do not expose. Until both are in hand, the empirical reconstruction
here stands and its caveats above apply. When the tables are loaded, this should be rebuilt as the lookup and
retired — the empirical fit is a stopgap for want of the tables, not a preferred design.

Also worth knowing: the LCM was 1.00 in the program's first year and has been 1.64 since, so any rate change
spanning that boundary reads as a price rise that is really the introduction of the expense load. And a
positive manual premium plug appears on a few dozen transactions — judgement-priced premium sitting *beside*
a rate that is already correct. It is flagged, not folded in.

**Renewal rate change: this is the book where the exposure unit itself moves at renewal**, and it is the
one thing to understand before reading a rate change here. It is the largest single reason a pair is
withheld. Three shapes, and they are not the same problem:

- **The unit genuinely changed.** A risk rated per location one year and per activity day the next, or per
  attendant one year and on gross sales the next. The recorded quantity moves by orders of magnitude —
  nineteen locations becoming seven hundred activity days — while premium moves a few per cent. The ratio
  of the two rates is arithmetic, not price, and it is refused. This is the large group.
- **The unit was only renamed.** The program records the same headcount under several interchangeable
  names — each, each member, each participant, each attendant, admissions — and a policy's label can move
  between its own two terms while the count stays put. These **are** published, flagged with
  `basis_comparable_by_renamed_unit` so they can be excluded, because the equivalence of the two labels is
  inferred by us rather than stated by the source. The inference rests on the recorded quantity holding
  still, not on the resulting rate change looking sensible; the threshold is in
  [publishing_a_rate.md](publishing_a_rate.md).
- **A single basis becoming a blend of several.** Refused, as everywhere else.

Worth knowing that the relabelling turned out to be **price-neutral**: those pairs move at the same median
as the rest of the book and on a tighter spread, so nothing was being taken or given behind the rename.
Do not read a terms-adjusted change on that subset — few of them carry one and the limits factors
underneath are unstable.

---

## Senior Living Healthcare Professional Liability

**KBSI.** Professional liability for senior living and skilled nursing facilities — the subsidiary's
primary line.

**Priced on a weighted bed count.** Beds are counted at up to five levels of care and weighted to a
common unit: skilled nursing 1.00, intermediate and memory care 0.80, assisted living 0.60, independent
living and adult day care 0.35. Rating uses **occupied**, not licensed, beds. The result is a *skilled bed
equivalent*, and it is the exposure.

This program was initially reported as having no exposure at all. It does — it simply records it
somewhere no other program does, so a search built on the GL pattern finds nothing. Worth remembering as
a general caution about this source.

**The trap: three of its factors run the opposite way to every other program.** The deductible factor is
a *multiplier* here rather than a subtracted credit; the limit factor *reduces* the price rather than
loading it; and neutral terms sit at the *top* of every factor range rather than the bottom. The
practical consequence is that normalizing restates senior living rates **up** where it restates every
other program's **down**. A senior living normalized rate placed beside a GL one is meaningless.

**One number here is derived, not sourced.** The source records *whether* an underwriter applied a
schedule modification but not by how much, and that gap was searched exhaustively before being accepted
as a vendor limitation. The modification is reconstructed as the residual between the charged premium and
the priced premium, is labelled as derived, and is a summary of several separate per-location judgements —
read it with the location count beside it, and never present it as a source value.

**The ladder:** all three rungs. This is the only program with a genuine recorded manual premium.

**Renewal rate change:** covered, and it publishes a terms-adjusted change because the factor stack is
recorded. The book is per skilled bed on both terms, so pairs compare cleanly. The one caution is the
normalization direction above — a senior living normalized rate change must never be pooled with a general
liability one; the model only publishes it when both terms share a normalization basis.

---

## Business Auto

**All three subsidiaries** — the only cross-subsidiary line.

**Priced per vehicle, per coverage.** A single vehicle generates a liability row, a collision row, a
comprehensive row, a medical payments row and so on, each an independent calculation with its own base
rate and **its own formula**. That is why auto carries roughly twice as many factor types as GL.

Unlike GL, the base rate here is already a chargeable price per vehicle — there is no bureau loss cost
and no LCM, so **there is no filed-rate benchmark on this line**. The rate is built by summing the
engine's own recorded premium per vehicle rather than reconstructing the chain.

**The trap: two source labelling collisions that both silently corrupt totals.** Two premium type codes
share the name "Transaction" — one is premium and the other is tax and surcharge, and auto is the only
line carrying the latter, so a GL sanity check never catches it. Separately, two placeholder codes
display identically, one additive and one a restatement that double-counts. Both are handled by code, and
both are held by `assert_pds_type_ids_unchanged` and
`assert_transaction_premium_excludes_tax_surcharge`.

Two more: several auto "factors" are **dollar amounts sitting among the multipliers**, so any generic
multiply-everything logic corrupts the line. And an auto cancellation deletes its whole vehicle schedule,
so the exposure vanishes while the coverage rows survive — **auto is the only line where a cancellation
publishes no rate.**

**The ladder:** charged rate and manual rate. **Normalized rate declined**, deliberately: one deductible
factor's behaviour is unresolved on roughly 15% of liability premium, which is too much premium to guess
on. Held by `assert_auto_publishes_manual_but_not_normalized`.

**Renewal rate change:** covered. Auto is per vehicle within a program, so a customer's auto renewal
compares cleanly. There is no terms-adjusted change — auto is not normalized — so read
`terms_changed_at_renewal` (off the recorded limits and deductible) rather than a magnitude. A cancelled
term has no rate, so it simply drops out of the comparison.

---

## Umbrella / Excess

**TPM, RSI and KBSI.** Liability cover sitting above a primary policy.

**This is the one line with no exposure of its own anywhere in the source** — no class codes, no exposure
limits, no risk schedule. It is priced as a **percentage of the underlying primary premium**, so the
published measure is a percentage rather than a price per unit. Typical relativities run from around 13%
on senior living to around 50% on the largest contractors tower.

The pairing to the primary policy is the whole job, and it is done on the account number, falling back to
the named insured, recording which fired. An excess policy is matched to the primary **in force during
its term** — and a mid-term excess policy is priced against the full annual underlying premium, because
the layer sits above the primary's annual aggregate.

Two rules on the denominator. It is the **primary casualty line only** — General Liability, or
professional liability where that is the program's primary. **Business auto premium is carried as a memo
column and stays out**, because including it would move the relativity only on the accounts that happen
to place auto with us.

**Premium decomposes by attachment layer, not by coverage** — one row per layer of limit — and the price
chain exists only on the first layer, so a large minority of the line's premium carries no factor detail
at all.

**The trap: the limit factor identifies which excess product this is, not what the customer bought.** It
is essentially constant within each tower. Dividing it out would erase the distinction between two
genuinely different contracts, which is worse than not normalizing at all.

One thing not to "fix": almost every excess term reads as having no deductible, and that is correct.
Excess attaches above a retention rather than carrying a deductible — the right answer is *not
applicable*, not zero.

**The ladder:** the relativity only. **No manual and no normalized rate**, and both absences are
properties of the product rather than gaps in the analysis.

**Renewal rate change:** covered — the change in that relativity year over year, which is a real price
signal. Because pairing is within one line, the two Contractors towers (`Excess 3X2`, `Excess 5X5`) never
compare against each other, even though they share one comparison group. No terms-adjusted change, since
there is no normalized rate.

---

## Legacy Velocity (pre-2025)

Not a program but the **platform that PDS replaced**, and only Contractors and Habitational have any
legacy population.

Velocity has one generic exposure column, no basis label, no bureau loss cost and no factor stack — but
it does record its own rate. So for legacy terms **rate is available and rating mechanics are not.**

**The trap: a legacy term looks like a PDS term in `fact_pds_policy_term`, and is not.** Around 250 of the
unpriced terms are legacy, and no charged-rate absence reason exists for any of them, because
`fact_pds_policy_exposure_rate` holds PDS transactions only. Check `source` before concluding anything about a
missing rate or factor.

The normalized rate is therefore NULL on a legacy term by default, with the reason stated, and legacy renewal
pairs carry a flag saying whether the recorded limit or deductible changed — the signal that matters without
fabricating a magnitude.

**There is now one switch that overrides that, and it is off.** `fact_pds_policy_term` can publish an
*estimated* normalized rate on legacy general liability terms. It estimates the legacy term's **own** baked-in
factor from two sources in precedence order: the factor **recorded on that insured's own successor** where they
bought identical limits and deductible a year later (`int_pds_legacy_terms_factor_anchor`, reaching 113 of the
311 published legacy terms at a median 0.10% error), and a **group average** over program, state, limits and
deductible otherwise (`int_pds_estimated_terms_factor`). Neither substitutes another policy's factor, and
neither changes the neutral standard the rate is restated to.

The long-standing objection to overlaying PDS factors — that they are from the wrong era — is handled by
restricting the estimate to terms incepting inside a recent window. The second objection, that the factors are
not determined by the limit, turned out to be wrong: they are largely determined by the limit and the deductible
together, once state is allowed to move the limit leg. What remains true is that the group average is a **fitted
group figure and not the factor anyone charged**, which is why the switch exists, why it is off by default, and
why the successor's own recorded factor takes precedence wherever it is available. See
[publishing_a_rate.md](publishing_a_rate.md) § 2.

One consequence to expect rather than be surprised by: on an anchored term both sides of a renewal comparison
carry the same factor, so the normalized rate change equals the raw rate change. That is correct — the insured's
terms did not move — and removing that spurious difference is the point.

One consequence for reporting: both systems label Contractors exposure *composite receipts*, so a
composite-rated term compares across the platform change and the exposure agrees to the dollar. What is
refused is a composite-rated term paired against a **class-code-rated** one. In PDS a Contractors GL policy
rated on gross sales or payroll is class-code rated without exception, while every composite-rated policy is
on composite receipts — so the basis separates the two rating methods. A rate built up from class codes is
not the same measure as a negotiated composite rate even quoted in the same unit, so those pairs are
suppressed with a stated reason. That is refusal on rating method, not on naming.

**The ladder:** charged rate only, because the source carries no factor stack to divide out. No manual rate
under any circumstances — nothing in the legacy source stands in for an underwriting modification, and on a
composite policy the negotiation *is* the judgement, so there is nothing separable to divide out even on PDS. A
normalized rate only if the estimated-normalization switch is on, and then it is estimated — measured on the
term's own successor where one exists on identical terms, otherwise fitted from a group average.

### Four things about the legacy rate that are settled, and should not be re-litigated

These were each measured against the platform's own recorded rate, `comp_rate_`, which is the only external
validation the rate derivation has anywhere in this project.

**The rate is not annualized, on any term length.** A term's premium and its exposure cover the same period,
so their ratio is already independent of term length. Scaling the numerator to a year without scaling the
denominator beside it restates one side of a ratio whose sides already agree. The evidence: `comp_rate_`
never annualizes anything, and the unadjusted figure reproduces it to a median ratio of **exactly 1.0000**
on terms running longer than a year, where an annualized figure reads about **0.92**. On a 547-day term the
unadjusted rate reads 63.00 against a recorded 60.00; annualized it reads 30.76.

**A mid-term excess placement needs no span adjustment either.** The excess relativity is the one measure
whose numerator and denominator come from *different policies*, whose terms can genuinely differ in length —
1,483 of 1,679 excess terms match a primary of identical span, and 95 are mid-term placements running a
median 232 days against a primary's 365. Those 95 still need nothing done to them, because a mid-term excess
is **rated and priced fully annual**: its premium is already a whole-year amount and compares directly to a
full-term underlying. Scaling it would inflate a figure that is already annual.

**A cancellation is a restatement, not a short term.** The platform rewrites `expiration_date` to the
cancellation date on every ledger row of a cancelled term, so the stored span measures how long the policy
*survived* rather than the term it was written for. Never read that span as a term length. Separately, the
ledger carries a credit returning the unexpired premium, and that credit is **excluded** from the rate
numerator: including it reports what the term retained rather than what it was charged, and reads about a
quarter below the price actually struck.

**One open question.** The 26 short, non-cancelled terms disagree with the recorded rate under *either*
definition — 0.79 unadjusted, 1.12 annualized, with only 11 and 3 of 26 respectively landing within 2%. That
is a question about how the platform booked part-year premium, not about annualization, and it is not
answered. Read a short legacy term's rate with that in mind.

---

## Exposure bases and comparison groups

Every exposure basis resolves to one of five **basis types**, which is what determines whether two rates
can be compared. The mapping lives in the `exposure_basis_type` macro and
`assert_all_exposure_bases_are_mapped` fails the build when a new basis appears at source — so a new unit
surfaces as a broken build rather than a silent gap.

| Basis type | Meaning | Examples |
|---|---|---|
| **Currency** | A dollar amount of exposure. The unit is objectively defined. | Gross Sales, Payroll, Composite Receipts |
| **Area** | Square footage. | Area |
| **Count** | A count of business objects. **The object differs by program.** | Units, Beds, Vehicles, Members, Admissions, Acres |
| **Flat** | A fixed charge. **No per-unit rate exists at all.** | Flat Charge |
| **Relativity** | A ratio of two premiums — a percentage, not a price. | Underlying Primary Premium |

From which the comparison rule follows:

- **Currency and Area** — the unit means the same thing everywhere, so group by basis and divisor.
  Cross-program comparison is legitimate.
- **Count and Flat** — the unit is a business object, so group by basis, divisor **and program**.
- **Relativity** — the attachment structure differs by program, so group by basis **and program**. The
  five programs' relativities are five different products, not five estimates of one number.

All of it collapses into `rate_comparison_group`.

### Where a policy mixes bases

| `basis_status` | Meaning | Rate published |
|---|---|---|
| `single` | One basis. | That basis's rate. |
| `blended` | Several bases sharing the same type *and* divisor. | Summed exposure, blended rate. |
| `dominant_basis` | Several bases of different kinds, one holding ≥75% of premium. | The dominant basis's own rate, unchanged — **never a blend across kinds**. |
| `not_comparable` | Several kinds, none dominant. | None. Read per basis from `fact_pds_policy_exposure_detail`. |
| `no_exposure` | No usable exposure. | None, with a stated reason. |

**`basis_status` is not the whole publication rule, and reading it as though it were is a trap.** Two things
it does not cover. First, a `dominant_basis` policy still has its rate withheld where the dominant basis
covers less than 75% of the whole **policy** premium — the same 75% applied a second time, against a
different denominator. Second, a rate is also withheld for reasons that have nothing to do with mixing
bases at all: a flat charge, a negotiated premium, hired-and-non-owned auto only, or an unmatched excess
policy. Those come from `rating_method`, not from `basis_status`. The full decision order and every reason
are in [publishing_a_rate.md](publishing_a_rate.md).

### One basis can be two names for the same thing

Comparability is otherwise decided entirely by `rate_comparison_group`, with one deliberate exception, and
it applies only to the renewal comparison rather than to grouping generally. Sports & Entertainment uses
several interchangeable names for the same headcount, so a policy can appear to change its rating unit
between its own two terms when only the label moved. `fact_pds_policy_renewal_rate_change` allows such a
pair and marks it; the rule and the reason it is scoped to a pair rather than folded into the comparison
group are in [fact_models.md](fact_models.md) and
[publishing_a_rate.md](publishing_a_rate.md).

Note the asymmetry, because it is the point: two terms of **one policy** whose label moved while the count
held still are almost certainly the same unit. Two **different customers**, one rated per member and one
per admission, are not — and nothing here licenses averaging those together.

---

## Coverage summary

| Program / line | Charged rate | Manual rate | Normalized rate |
|---|---|---|---|
| Habitational GL | yes | absent — none recorded on the program | yes, the reference case |
| Senior Living HPL | yes | yes | yes |
| Sports & Ent. GL | yes | yes | yes, on issuing transactions |
| Liquor | yes | yes | yes |
| Contractors GL | yes | no — the negotiation is the judgement | yes, from the build-up rows |
| Business Auto | yes | yes | **declined** — one factor unresolved |
| Umbrella / Excess | yes (relativity) | no — none exists | no — would erase the product |
| Legacy Velocity | yes | no | no — no factor stack in source, unless estimated normalization is switched on; then anchored on the term's own successor where possible, group-fitted otherwise |

Every `yes` in the normalized column is additionally subject to the terms-coverage gate, which is why a
policy on a program marked `yes` can still carry `terms factor covers too little of the rated premium`. See
[publishing_a_rate.md](publishing_a_rate.md).

**Known partial coverage.** Auto is verified on Liability only — which is the large majority of auto
premium, but each other coverage composes a different formula. The senior living retention is recorded at
source but not yet surfaced onto the term. Habitational's missing judgement factor is an open
underwriting question rather than a modelling gap.
