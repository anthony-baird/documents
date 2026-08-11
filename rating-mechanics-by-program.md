# How rating works, by program and line

**Purpose:** document how premium is actually calculated in PDS for each program and line of business,
so that exposure, rate and rate-change models are built on the real mechanics rather than on a shape
inferred from one program.

**Status:** TPM Habitational GL, RSI Sports & Entertainment GL, TPM Contractors GL and Business Auto are
complete and verified. KBSI Senior Living HPL is documented in §4.5 but has not been read back against the
built design. Umbrella/Excess is documented in §4.6 — its first attachment layer reproduces exactly, and
the line is priced on a stored equation rather than on an exposure.

**Method:** every formula below was reconstructed from worked examples, then validated at scale against
landed data. Accuracy figures are stated per program. Where a formula does not close, that is recorded
rather than smoothed over.

---

## 1. Read this first — five traps

These caused wrong answers during discovery and will do so again.

**1. Use `annual_premium`, not `written_premium`.** `written_premium` on an endorsement is the
*increment*, not the annualised amount. Reconstructing a rating formula against `written_premium`
produces ratios that vary randomly across rows carrying identical factors, which looks like a broken
formula rather than a wrong measure. Validate against `Issue` transactions first.

**2. Factor names come from `stg_pds_factor.datas_ource_field`.** `type_description` is blank or
cryptic on most factor rows (`SBE RATE`, `BASE`, `CM`). `datas_ource_field` carries the rating engine's
own field name (`loss_cost_0`, `base_rate_0`, `lcm`, `ilf`, `deductible_0`, `rmf`).

**3. `FACTOR.transaction_id` is a storage/version scope, not a policy transaction.** The table holds
~3.2M rows across only 77 distinct values. Joining it to `dim_transaction.transaction_id` fans out
catastrophically and returns plausible-looking totals. Reach factors through
`PREMIUMFACTOR.premium_id`, and reach the transaction through
`dim_premium.transaction_key_coalesced`.

**4. A factor's presence in `FACTOR` does not mean it was used.** The table contains rating-table
entries that are never attached to a premium. Measured on the raw table, `Ded` has a median of −0.12
and a minimum of −0.699; restricted to rows actually linked through `PREMIUMFACTOR`, it is always
positive. Only ever measure factors that are attached to a premium row.

**5. Not every row in `FACTOR` is a multiplier.** Some are dollar amounts (`Adj` −601 to +123 and
`Adc1` 1.0–41.8 on auto; `Amount` 15/50/100 for rental reimbursement), some are counts (`SBE RATE` is a
bed count), and some are subtotals of the calculation itself (`BASE PREM`, `UNADJ PREM`, `SubPrem` —
their `datas_ource_field` values are literally `subtotal_N`). Generic "multiply every factor" logic
produces nonsense. Check `datas_ource_field` and the value range before treating anything as a
multiplier, and note that the same factor can be a multiplier in one program and a signed credit in
another — `Ded` is multiplicative on KBSI and a credit fraction on GL.

---

## 2. Factor dictionary

Shared across the class-code-rated GL programs. Ids resolve against `stg_pds_type.type_id`.
**Business Auto and KBSI Senior Living HPL each carry their own factor set** — see §4.4 and §4.5. Do not
assume a factor id means the same thing across lines: `Ded`, `ILF` and `BASE` all behave differently.

| Factor | id | `datas_ource_field` | Definition |
|---|---|---|---|
| **BASE** | 1666 | `loss_cost_0` | **Bureau loss cost.** Expected losses per `PER` units of exposure for the class code and state. Pure premium — no expenses, no commission, no profit. This is ISO's number, not ours. |
| **LCM** | 1804 | `lcm` | **Loss Cost Multiplier.** Converts a loss cost into a chargeable rate: expenses, commission, profit. This *is* our filed pricing decision. |
| **CODEV** | 1715 | `codev` | **Company Deviation.** A filed departure from bureau rates. Constant `1.0` in every program measured — not in use. |
| **PREMIUMBASE** | 1872 | `base_rate_0` | **The charged base rate**, derived: `loss_cost × LCM × CODEV`. Not an independent input. |
| **PER** | 1860 | `per` | **Exposure denominator.** `1` (per unit), `100`, or `1000` (per $1,000 of receipts / payroll / area). This is the rate divisor. |
| **ILF** | 2047 | `ilf` | **Increased Limit Factor.** Loads basic-limit premium up to the limit actually purchased. |
| **Ded** | 2014 | `deductible_0` | **Deductible credit as a decimal fraction — NOT a multiplier.** `0.225` means a 22.5% credit, and it applies to the **basic limit layer only**. See §3 step 4. |
| **RMF** | 411 | `rmf` | **Rate Modification Factor** — schedule rating / individual risk modification. The underwriter's judgement, recorded as a number. |
| **REDUCT** | 2118 | `eqn_element_10` | A plain **multiplier**, applied in the same product as `RMF`. Named for a reduction and mostly is not one — it takes 13 distinct values on RSI, from `0.70` to `22.64`, with the small integers 2/3/4/5 the most common. Inert at `1.0` on all but 42 issuing rows. |
| MED PAY | 2060 | `eqn_element_0` | Medical payments element. |
| PER&ADV | 2100 | `eqn_element_1` | Personal & advertising injury element. |
| CLAIMS MADE | 1998 | `eqn_element_2` | Claims-made element. |
| PREM ADJ | 2111 | `eqn_element_3` | Premium adjustment element. |
| Damage | 2012 | `eqn_element_4` | Damage-to-premises element. |
| LEAD FAC | 2052 | `eqn_element_11` | Lead factor. |

The six `eqn_element_*` factors are present on essentially every rated row and **inert at `1.0`** in both
GL programs. They are placeholders for coverages not written in these programs. Carry them, do not
assume they stay at 1.0 in a program not yet examined.

Exposure itself is **not** a factor: it is `LIMIT.limit_amount` where `limit_type_id = 549`
(`Exposure`), reached through `PREMIUMLIMIT`. `LIMIT.rating_basis` carries the divisor where populated,
and agrees with `PER`.

---

## 3. Class-code rated GL — the general formula

Applies to **TPM Habitational GL** and **RSI Sports & Entertainment GL**.

```
annual premium = loss_cost × LCM × CODEV          -- the charged base rate
               × (exposure ÷ PER)                 -- how much risk
               × (ILF − Ded)                      -- what they bought
               × RMF                              -- what we decided
               × REDUCT × MED PAY × …             -- normally 1.0
```

Premium is calculated **per class-code coverage**, at two premium types: `334` (Premises/Operations)
and `336` (Products/Completed Operations). Each is a separate rated row with its own factor set.

### Step 1 — start from the bureau

ISO publishes a loss cost per class code per state. Habitational class `60010` (apartments) carries a
loss cost of `67.615` per unit.

### Step 2 — load it into a chargeable rate

Multiply by our filed LCM and company deviation. `67.615 × 1.48 × 1.0 = 100.0702`, which is exactly the
value stored as `PREMIUMBASE`. **This split is the single most important thing in this document for
rate-change purposes:** the loss cost is the market's view of the risk, the LCM is our view of what it
costs us to write it. "Rate went up" means something completely different depending on which moved.

### Step 3 — apply exposure

`rate × (exposure ÷ PER)`. Apartment units use `PER = 1`; receipts and square footage use `PER = 1000`.

### Step 4 — adjust for limits and deductible, and note the structure

The deductible credit applies **only to the basic limit layer**. Increased limits load on top of it and
are **not** discounted:

```
premium = basic × (1 − Ded)  +  basic × (ILF − 1)   ≡   basic × (ILF − Ded)
```

So `ILF 2.35` with a 22.5% deductible credit gives a combined term of **2.125**, not
`2.35 × 0.775 = 1.821`. Treating `Ded` as a multiplier is wrong and is what defeated earlier attempts
to reproduce the chain. Verified against worked examples:

| ILF | Ded | `ILF − Ded` | observed premium ÷ base premium |
|---|---|---|---|
| 2.35 | 0.225 | 2.125 | 2.1250 |
| 2.99 | 0.225 | 2.765 | 2.7650 |
| 2.99 | 0.137 | 2.853 | 2.8530 |
| 2.58 | 0.108 | 2.472 | 2.4718 |
| 2.35 | 0.137 | 2.213 | 2.2130 |

Where `Ded` is absent, the term is simply `ILF` (equivalent to a zero credit) — this reconstructs
correctly on the 1,319 RSI rows with no deductible factor.

### Step 5 — the underwriter deviates, or does not

`RMF` is schedule rating: credit or debit for risk characteristics the class code does not capture.

### Verification

Reconstructed premium against `annual_premium` on `Issue` transactions:

| Program / premium type | rows | median actual ÷ predicted | within 1% | within $1.00 |
|---|---|---|---|---|
| TPM Habitational `334` | 464 | 1.0000 | **99.8%** | 94.4% |
| RSI `334` | 5,058 | 1.0000 | **94.4%** | 86.4% |
| RSI `336` | 1,998 | 1.0010 | 48.9% | **82.9%** |

RSI `336` is weak on the percentage test and sound on the dollar test because median premium on those
rows is **$7 to $41** — a one-dollar rounding difference exceeds 1%. Read the dollar column for
products/completed-ops rows.

Broken out by factor availability on RSI `334`: rows with a full factor set reconstruct at 98.8% within
5% (n=3,697), and rows with no deductible factor at 98.0% (n=1,319).

---

## 4. Program-specific behaviour

### 4.1 TPM Habitational GL

- **Exposure bases:** apartment units (`PER = 1`), plus square footage, discrete features and tenant
  receipts (`PER = 1000`). Units carry the overwhelming majority of premium.
- **LCM:** effectively constant at **1.48** (three distinct values across the program).
- **`RMF`: exactly 1.0 on all 464 rows — zero credits, zero debits.** Schedule rating is not used.
- **Consequence:** every dollar of Habitational rate movement comes from the bureau loss cost, the
  exposure, or the terms purchased. None of it comes from underwriting judgement, because none is
  recorded.
- **No products/completed-operations premium exists on this program.** Every rated dollar is `334`:
  $360.20M across 5,578 coverages, against **zero `336` rows** (RSI carries 1,998). So the §3
  verification above is complete for Habitational rather than covering one of two premium types, and
  §5 item 3's rule that products rates need an absolute tolerance has nothing to apply to here.
- **Term drift is large, and it very nearly cancels an opposite move in price.** Medians on issuing
  `334` rows:

  | | 2025 | 2026 | change |
  |---|---|---|---|
  | loss cost | 108.43 | 86.40 | **−20.3%** |
  | LCM | 1.4800 | 1.4800 | flat |
  | **price** (loss cost × LCM) | 160.48 | 130.70 | **−18.6%** |
  | **terms** (ILF − Ded) | 2.2130 | 2.6770 | **+21.0%** |
  | **achieved rate** | **364.42** | **346.18** | **−5.0%** |

  Read the achieved rate alone and Habitational pricing looks roughly flat. It was not: **the filed
  price fell nearly a fifth**, and insureds buying more limit or accepting lower deductibles masked it.
  This is the strongest concrete argument for separating terms from price — not merely that the terms
  moved, but that the two movements are of similar size and opposite sign, so the net is small and
  uninformative.
- **All of it is readable from source.** Loss cost, LCM, ILF and the deductible credit are complete on
  every rated coverage, so separating price from terms on this program is a source read rather than a
  derivation. Contrast Contractors in §4.3, where the same factors exist but sit on shadow build-up
  rows that the negotiated rate does not honour.
- **The published rate is verified against the engine's own stored chain.** `rate = base_rate ×
  (ILF − Ded) × RMF` holds on **5,572 of 5,572** rated coverages within $1.00, worst gap $0.92, across
  every transaction type including cancellations — asserted by
  `assert_class_code_rate_reproduces_stored_base_rate`. The stored base rate itself reproduces
  `loss_cost × LCM × CODEV` on 464 of 464 issuing rows.

### 4.2 RSI Sports & Entertainment GL

- **Exposure bases:** many — the program carries far more distinct bases than any other.
- **LCM:** **1.64** from 2023 onward; **1.00** in 2022, i.e. loss cost was charged unloaded in the
  program's first year. Worth knowing before reading a 2022→2023 rate change.
- **`RMF` is real, and its use is trending sharply:**

  | Policy year | 2022 | 2023 | 2024 | 2025 | 2026 |
  |---|---|---|---|---|---|
  | rows with a credit | 13.7% | 21.4% | 36.3% | **51.7%** | **52.6%** |
  | rows with a debit | 2.4% | 1.2% | 8.7% | 11.1% | 13.1% |
  | median RMF | 1.000 | 1.000 | 1.000 | 0.980 | 0.985 |

  Range `0.195` to `5.55` across 110 distinct values. Over half of RSI's rated exposure now carries a
  schedule credit, against one row in seven in 2022.
- **Consequence:** for RSI the "how far off manual are we" question needs **no reconstruction at all** —
  read `RMF` directly.
- **Term drift:** `(ILF − Ded)` moved **1.000 → 1.733** across 2022–26 (median on issuing `334` rows).
- **Premium completeness — restated.** The original reading of this, that premium types outside the
  rating enumeration were not captured, is **no longer true**: every premium type on the program is now
  named, and the written-premium identity closes at **$0.00 on all 2,029 rated transactions** with zero
  failures. What remains is not missing premium, it is premium that is honestly outside a rate. 520 of
  those transactions still sit below 0.90× policy premium, and the gap is named: **$62.1M of `@P`**
  coverage-builder endorsements, flat charge, and the manually rated plug. Rate-level leakage —
  rateable premium behind an exposure recorded as zero — is **$48.1K, 100% attributed**.
- **The plug on RSI is positive, and is not netted into the rate.** 42 rated transactions carry a
  manually rated figure of at least 10% of the rate numerator, median **2.76×** it, and **not one is
  negative.** That is the opposite of the Contractors back-solve in §4.3: there a negative plug corrects
  a build-up that was never the price, here a positive plug is judgement-priced premium sitting *beside*
  a rate that is already correct. Netting it in would put premium into the numerator with no exposure
  under it. They are flagged (`is_manual_premium_material`) and left alone.

#### The published rate is the rate the system applied

Verified two ways, on issuing transactions.

**Directly, rate against rate.** On single-basis, single-coverage policies whose stored chain is
complete, the published rate reproduces `base_rate × (ILF − Ded) × RMF × REDUCT` from the factors:
**169 policies, median ratio 1.000000, p05 0.999938 to p95 1.000318** — 96.4% within 10 basis points,
98.8% within 1%. The residual is cent-rounding on the stored premium, not a difference of method.

**Indirectly, premium against premium**, which covers the whole population rather than the clean
subset. Reconstructing annual premium from the factor stack over every `334` / `336` / `332` row on an
issuing transaction:

| | premium rows | annual premium | |
|---|---|---|---|
| stored chain reproduces the charged premium | 8,717 | **$158.4M** | 76.8% |
| no stored base rate at all | 1,538 | $38.9M | 18.8% |
| stored chain does **not** reproduce it | 388 | $9.0M | 4.4% |

Because exposure enters the reconstruction linearly, a chain that closes to the cent is also proof that
**the exposure and the divisor we publish are the ones the engine divided by**.

Read against the §3 verification table rather than instead of it: §3 measures how well the formula fits
where a formula exists, and reports 94.4% on `334`. The table above asks the different question of how
much of the program's money that covers, so it keeps the rows with no factors in the denominator instead
of excluding them.

#### So there is no source rate to read instead — deliberately, not for want of looking

Nearly a fifth of RSI's issuing premium carries **no stored base rate whatsoever**: the rate was
overridden at entry and PDS keeps the resulting premium and nothing else. Across all transaction types
that rises to **45.1%**, because endorsements largely do not restate the factor stack. On a further 4.4%
a chain is stored and the premium charged does not follow it.

This is the Override Rate situation, and it settles the design question for the program: the charged
premium is the only measure present on every row, so the rate stays **derived as charged annual premium
÷ exposure × divisor**. A published rate read from the stored base rate would be null on a fifth of the
book and wrong on a further twentieth of it.

The two inputs beneath that derivation are, by contrast, **fully sourced and never inferred**:

- **Exposure** is `LIMIT.limit_amount` at type 549, and the reconstruction above proves it is the figure
  the engine used.
- **The divisor** is source on every single row — the limit's own `rating_basis` on general liability
  class codes (36,070 coverages), the `PER` factor on Liquor Liability (4,282), where `rating_basis` is
  NULL throughout. Where both exist they agree on **68,658 of 68,658** rows, with no disagreement
  anywhere in the program, so the liquor fallback reads the same number by its other route.

`assert_class_code_rate_reproduces_stored_base_rate` therefore **cannot be extended to RSI**, and the
reason is a property of the program rather than a gap in the test. Of 18,537 coverages carrying a
complete stored chain, **3,431 (18.5%, $246.5M)** have a chain that does not reproduce what was charged.
Habitational passes that test because its override population is zero.

**The 25 exposure bases are not a nuisance to be normalised away.** They carry genuinely different
prices — `Each Attendant` at a median $0.38 and `Each Event` at $435.60 — which is why the
dominant-basis and not-comparable tiers exist and why 103 transactions publish no rate rather than a
blended one. Every unrated RSI transaction carries a stated reason; none is silently dropped.

### 4.3 TPM Contractors GL — composite rated

Contractors is **composite rated**: a single negotiated rate applied to receipts, rather than a
class-code build-up priced factor by factor. The book is predominantly **New York Free Trade Zone**
business — FTZ is exempt from filed rate and form requirements — and is priced on loss experience and
venue. Measured: NYFTZ is flagged on **$468M of the $718M** composite book, and that segment is
**93.7% New York**.

The difference from §3 is structural, not a data gap. But it is a difference in *where the rating
decisions are recorded*, **not** a difference in which decisions are made: limits, deductibles and class
of work all drive the premium here exactly as they do elsewhere.

#### The formula

```
annual premium = composite_rate            -- one negotiated price for the whole risk
               × (exposure ÷ 1000)         -- how much risk, per $1,000 of receipts

  composite_rate is NEGOTIATED, but it is DERIVED from a class-code build-up
  that carries the limits and deductibles:

  composite_rate = Σ[ loss_cost           -- the bureau's view of the risk
                     × LCM                -- our expense and profit load
                     × (exposure ÷ PER)   -- how much risk, per class code
                     × (ILF − Ded)        -- what they bought
                     × RMF ]              -- normally 1.0 on this program
                   ÷ exposure × 1000      -- collapse the build-up into one rate
```

So the same rating elements as §3 are present — they sit **one level up**, inside the derivation of the
rate, rather than being applied to the premium row.

**Step 1 — the class-code build-up runs.** PDS rates the policy conventionally, per class code, using
exactly the §3 chain including `ILF` and `Ded`. These rows land as premium types `334` and `336`.

**Step 2 — the build-up is collapsed into one rate.** Total build-up premium ÷ exposure × 1000 becomes
the composite rate, stored as coverage extended data type **14399**. Verified at 100% (see below).

**Step 3 — the rate is negotiated.** The stored composite rate is what the manual build-up implied; the
rate actually charged is agreed with the broker against loss experience and venue. The two diverge
widely and unsystematically — this is the step that makes Contractors different, and it is where FTZ
exemption from filed rates matters.

**Step 4 — premium is the charged rate applied to receipts.** `CMPRAT` for a single composite rate;
`CMGP*` for multiple composite groups, one row per group.

#### Where each element lives

| Element | Where | Detail |
|---|---|---|
| Charged premium | `CMPRAT` (single composite), `CMGP*` (multiple composite) | $681.3M and $234.9M respectively |
| Exposure (receipts) | `LIMIT` type 549 via `PREMIUMLIMIT` | Present on all 1,249 rows |
| Composite rate (as derived) | `COVERAGEEXTENDEDDATA` type **14399** `Composite Rate` | Exactly 1,249 coverages — one per composite premium row |
| Composite group name | `COVERAGEEXTENDEDDATA` type **14686** `Composite Group Name` | 439 rows (multiple-composite only), 40 distinct values naming the rating segment — `NY Rate`, `Sub Contracted`, `Steel Erection`, `Installation`, `Manufacturing`, `NYC Ops`, `GC Rate`, … |
| **Limits** | **LOB level** — `LOBLIMIT` → `LIMIT` | 1,017 rows = 810 composite + 207 multiple-composite, exactly one set per transaction |
| **Deductibles** | **LOB level** — `LOBDEDUCTIBLE` → `DEDUCTIBLE` | 1,016 rows, two types |
| `ILF`, `Ded`, `loss_cost`, `LCM`, `PER`, `RMF` | On the `334`/`336` build-up rows | Complete — these are the §3 factors |

**Limit types at LOB level:** Each Occurrence ($1M / $2M / $5M), Aggregate Limit ($2M / $4M / $10M),
Products/Completed Ops. Aggregate ($2M / $4M / $10M), Personal & Advertising Injury ($1M / $2M / $5M),
Damage To Premises Rented To You ($100k / $300k / $500k), Medical Expense ($5k / $10k).

**Deductible types at LOB level:** Premises/Operations and Products/Completed Ops, each taking one of
`$5,000 · $10,000 · $15,000 · $25,000 · $50,000 · $75,000 · $100,000 · $250,000 · $1,000,000 ·
$5,000,000 · N/A`. `N/A` is a legitimate stored value and parses to NULL, not zero.

#### What is absent at premium level

- **`CMPRAT` rows carry zero factor rows** — no loss cost, LCM, ILF, `Ded`, `RMF` or `PER`. No factor is
  applied *to the premium row*. The rating elements are on the build-up rows instead (Step 1).
- **`CMGP*` rows carry one factor**, type 2110 `Prem` / `adjusted`, which is a premium amount (median
  $400,000), not a rate factor.
- **No Contractors premium row of any type carries a `PREMIUMDEDUCTIBLE` link.** Deductibles are only
  ever at LOB level.
- **`LIMIT.rating_basis` is NULL on every composite exposure row**, which is why the rate divisor for
  this program cannot be read from source and is supplied as a constant of 1000.

There is therefore **no filed manual rate that the charged premium was struck against** — a consequence
of Step 3 plus FTZ exemption, not a recording failure.

#### Limits and deductibles do affect the premium — via the build-up

The absence of factors on `CMPRAT` must not be read as "limits and deductibles do not affect the
premium." They do, through Steps 1–2, and the mapping is deterministic. Measured across composite and
multiple-composite policies:

| LOB Premises/Operations deductible | `Ded` credit | | LOB Each Occurrence limit | `ILF` |
|---|---|---|---|---|
| $5,000 | 0.068 | | $1,000,000 | 1.88 |
| $10,000 | 0.129 | | $2,000,000 | 3.51 |
| $15,000 | 0.205 | | $5,000,000 | 4.99 |
| $25,000 | 0.188 | | | |
| $50,000 | 0.272 | | | |
| $75,000 | 0.331 | | | |
| $100,000 | 0.378 | | | |
| $250,000 | 0.542 | | | |
| $1,000,000 | 0.677 | | | |

The deductible credit rises monotonically with deductible size and the increased-limit factor with the
occurrence limit, exactly as in the class-code programs. `$5,000,000` deductibles carry no `Ded` factor
and `N/A` deductibles carry none either.

**Consequence — a term-adjusted Contractors rate IS computable.** The shadow class-code rows are
worthless as premium but they are a faithful record of the *terms* and of the manual rating inputs. The
blended term factor for a policy is the premium-weighted

```
term_factor = Σ(334 premium) ÷ Σ(334 premium ÷ (ILF − Ded))
```

and the term-adjusted rate is `charged composite rate ÷ term_factor`. Measured on `Issue` transactions:

| Policy year | policies | median charged rate | median term factor | median term-adjusted rate |
|---|---|---|---|---|
| 2025 | 117 | 74.00 | 3.3089 | 22.878 |
| 2026 | 103 | 61.86 | 3.2380 | 19.567 |

So the raw rate fell **16.4%** year on year while the term factor fell **2.1%** — meaning roughly two
points of the decline is insureds buying different terms rather than price movement, and the underlying
price movement is **−14.5%**.

Two caveats. Term adjustment does **not** reduce cross-sectional dispersion (IQR/median 1.151 → 1.119 in
2025, and 1.175 → 1.269 in 2026, i.e. slightly worse): Contractors' spread across policies is driven by
class of work, venue and loss experience, which the composite rate absorbs by design. And because the
composite rate is negotiated, `ILF`/`Ded` describe the *manual* build-up — whether a negotiated rate
honoured them in full is not knowable from the data. The adjustment is therefore sound for tracking
movement over time and should not be read as a statement about any individual policy's adequacy.

#### Step 2 verified — and why the build-up rows are not premium

The `334`/`336` rows on composite policies are live, are excluded by PDS from the transaction total, and
sum to figures far exceeding the whole book. Step 2 is now verified:

```
composite_rate (14399) = Σ(334 + 336 annual premium) ÷ exposure × 1000
```

Verified at **100%** of composite policies at `Issue` (median ratio exactly 1.0000; 100% within 1% in
every coverage-count bucket except one 4-policy bucket at 50%). **They are the manual build-up from
which the composite rate is derived — a rate derivation, not premium.**

They inflate for a mechanical reason. The numerator sums premium across *every* class-code coverage on
the policy, while the denominator is the composite coverage's single exposure figure. So the recorded
rate scales with coverage count:

| class-code coverages | policies | median recorded rate | median charged rate | charged ÷ shadow |
|---|---|---|---|---|
| 1 | 102 | 61.87 | 70.85 | 0.894 |
| 2 | 30 | 91.42 | 85.98 | 0.931 |
| 3 | 18 | 62.37 | 60.78 | 0.570 |
| 4 | 17 | 50.28 | 59.38 | 0.414 |
| 8 | 3 | 414.16 | 121.00 | 0.431 |
| 10 | 1 | 9,482.27 | 67.46 | 0.007 |
| 15 | 1 | 5,605.16 | 82.15 | 0.015 |

Consequences:

1. These rows must be **excluded from any premium summation** on composite policies. That is already
   the behaviour required for the premium identity to close.
2. The recorded `Composite Rate` is **only meaningful where the policy has a single class-code
   coverage**, and is inflated by a factor of roughly the coverage count otherwise. Do not read 14399
   as a rate without checking coverage count.
3. The **charged** composite rate is `CMPRAT premium ÷ exposure × 1000` and is the only defensible
   Contractors rate.

#### No deviation-off-manual measure — which is not the same as no term adjustment

Keep two questions apart:

- *"What did the terms contribute?"* — **answerable**, via the term factor above, because `ILF`/`Ded` are
  recorded on the build-up rows.
- *"How far off manual did we price?"* — **not answerable**, because Step 3 is a negotiation and the
  stored composite rate is a derivation artefact rather than a benchmark premium was struck against.

Even restricting to the 102 policies with a single class-code coverage, where the recorded rate is
internally consistent, the charged rate bears no stable relationship to it:

| Policy year | policies | median charged rate | median recorded rate | median ratio | p25 | p75 |
|---|---|---|---|---|---|---|
| 2025 | 54 | 82.00 | 74.46 | 0.6973 | 0.3681 | 1.8143 |
| 2026 | 48 | 62.18 | 41.30 | 1.1685 | 0.6093 | 3.3400 |

The interquartile range spans a factor of five and the median moves from 0.70 to 1.17 between
consecutive years. **The recorded composite rate is a derivation artefact, not a benchmark the premium
was struck against.** Any "deviation off manual" measure is therefore unavailable for Contractors, and
should not be promised for this program.

For Contractors the rate *is* the composite rate, and rate change is the movement in that rate at
renewal — term-adjusted where the build-up rows allow it, and compared within venue and class mix, since
the book is loss- and venue-driven.

#### Summary — what is and is not available for Contractors

| Measure | Available | Basis |
|---|---|---|
| Charged rate | **Yes** | `CMPRAT`/`CMGP*` premium ÷ exposure × 1000 |
| Limits and deductibles per policy | **Yes** | LOB level, `LOBLIMIT` / `LOBDEDUCTIBLE` |
| Term factor `(ILF − Ded)` | **Yes** | build-up rows; premium-weighted blend |
| Term-adjusted rate | **Yes** | charged rate ÷ term factor — for movement over time, not policy-level adequacy |
| Rating segment for multi-composite | **Yes** | `Composite Group Name` (14686) |
| Class of work / venue split | **Yes** | class codes on the build-up rows, state on the transaction |
| Rate divisor from source | No | `rating_basis` NULL on every composite row; constant 1000 supplied |
| Deviation off filed/manual rate | **No** | negotiated rate; no benchmark exists (FTZ) |
| Loss-cost vs LCM split of rate change | Partial | present on build-up rows, but the charged rate does not honour them |

#### Class-code rated Contractors policies are an artefact — exclude them

A minority of Contractors policies resolve to a class-code rating method. **These are not genuine
class-code rated business.** They are cases where underwriters back-solved a composite rate manually
because the platform's composite rating was not understood at the time. There is an internal plan to
endorse the composite rating plan back onto these policies once Insurity removes the block preventing
those endorsements.

Contractors, as a program, **never class-code rates**. Consequences:

- **Superseded — do not exclude them; correct them.** This originally read "exclude class-code-method
  Contractors policies from any Contractors rating or rate-change analysis". Once the mechanism below was
  understood, exclusion turned out to be the wrong remedy: the charged price *is* recoverable, because the
  manual row that back-solves it is recorded. Netting it in reproduces the policy's own charged premium
  exactly, so these policies now carry a correct rate rather than none. They still describe a workaround
  rather than a pricing approach, so read them with that in mind — every one is flagged.
- Their rated premium sums to **2.83×** policy premium ($279.3M rated against $98.8M policy premium on
  122 transactions), which is the same class-code inflation described above and further evidence the
  rows are a manual build-up rather than charged premium.
- Expect this population to shrink to zero once the re-endorsement completes. Any model keyed on it
  will silently change behaviour at that point.

##### How the back-solve is actually recorded — a large negative `MRC`

The mechanism is worth stating, because it is what makes the rate on these policies wrong while every
premium reconciliation on them passes. **The class-code build-up runs at full manual price, and PDS then
books a large negative `MRC` (Manually Rated Coverage) row to bring the total down to the negotiated
figure.** Worked example, `CZ01NYGL0068-02` at `Issue`:

| Row | Amount |
|---|---|
| `334` build-up | $13,654,769 |
| `336` build-up | $504,963 |
| **`MRC`** | **−$12,866,541** |
| = policy total | **$1,293,191** — exact |

So the policy total is right, and a premium decomposition that treats `MRC` as a named component closes to
the cent. What is wrong is the **rate**, whose numerator is the gross build-up: this policy prices at
**277.13** per $1,000 of receipts against a charged rate near **25.3**.

Measured across the population: **71 of 122** transactions publish a rate with the credit at **≥12.5%** of
the numerator (median 53%, max 91%), **−$199.2M** of credit in total. Separation from the rest of the book
is clean — composite Contractors tops out at exactly 10%, multiple-composite at 0%, and every other program
sits at a median of ≤1% — so the condition can be detected without naming the program.

Two things follow. **`MRC` sign matters and is not a curiosity:** a negative plug is a correction *to* the
rated premium, while a positive one is genuinely unrated premium that belongs outside the rate. And **the
same mechanism reaches other programs in the positive direction** — KBSI Senior Living has 44 rows whose
plug is a median 62% of the rate numerator, RSI Sports & Entertainment 41 GL rows at a median 2.8×. There
the rate is not overstated; it simply describes a minority of the price.

**On this population the plug is netted into the rate**, because the question it raises is already
answered here: the build-up is a working, not a price, so the premium charged is the build-up plus the
plug. The proof is that netting lands on the policy's own charged premium — the rate numerator moves from
1.86× annual policy premium to **exactly 1.0000** on every single-basis policy in the population. Two
worked examples, moving in opposite directions:

| Policy | Before | After | Charged premium |
|---|---|---|---|
| `CZ01NYGL0016-00` | 1,056,846 at **176.141** | 500,000 at **83.333** | 500,000 ✓ |
| `CZ01NYGL0023-01` | 527,000 at **13.175** | 630,000 at **15.750** | 630,000 ✓ |

The adjustment is applied at both the policy and the exposure-basis grain — spread pro rata across bases,
since the source records one figure against the policy and nothing to allocate it by — so the two grains
cannot drift. See the `is_back_solved_class_code` macro. **The rows stay flagged**: a price that is mostly
manual adjustment is worth knowing about even once the arithmetic is right.

**This is the exception, not the rule.** Everywhere else the plug is flagged and left outside the rate —
see `is_manual_premium_material` and `manual_premium_share` on `fact_pds_policy_exposure_rate`. Netting
generally would settle an underwriting question in a model, and would have to settle it differently in each
direction; withholding would hide a policy whose rate is the only price signal it has. **Every other
program still owes an answer on how its plug should enter the rating.** Note also that this carve-out is
expected to disappear: once the composite rating plan is endorsed back onto these policies they resolve to
composite, and the population empties.

### 4.4 Business Auto — all subsidiaries

Auto is **rated per vehicle per coverage** and then summed. It is the only line where the charged
premium is built from a large number of small independent calculations, and the only one where the
rated premium and the factor-bearing rows sit on **different premium types**.

#### Grain and where premium lives

| Level | Premium type | Detail |
|---|---|---|
| Policy | `Transaction` id **1449** | $372.99M (TPM Contractors) — no factors |
| Line of business | `Lob` id **1443** | $372.99M — no factors. Equal to the policy total **to the cent in every program**, which is the monoline premise stated as a live assertion |
| **Vehicle × coverage** | **`NULLVALUE` id 2399** | The calculated rows. **This is where all factors live, and this is what the rating models read.** |

**Two traps in this table, and both have bitten.**

**1. Filter the policy total by type ID, never by the type name.** Two different ids share the name
`Transaction`: **1449** is premium and **1572** is tax and surcharge. Reading the name sweeps tax into
premium — $2.71M on TPM Contractors auto alone, worst $46,811 on a single policy — and auto is the *only*
line carrying 1572, so a general liability check would never catch it. An earlier revision of this table
recorded the policy total as $390.00M against a line total of $387.23M and called the $2.77M gap a
rollup difference; it was the tax surcharge. **The two rollups do not differ at all.**

**2. The vehicle × coverage rows do not have a NULL premium type — they have a placeholder one, and
there are two placeholders, not one.** Both resolve to the display name `NULLVALUE`:

| Id | Description | What it is |
|---|---|---|
| **2399** | "Null Premium Nam" | The calculated per-vehicle, per-coverage premium. **Additive — this is the rating premium.** |
| **2272** | "Entry has no NAM. See Coverage Record." | A pointer that **restates premium already counted**. Including it double-counts by $1.95M. |

Neither id exists outside Business Auto anywhere in the book, so naming them is safe. Summing coverage
grain and excluding 2272 reproduces the line-of-business total on **1,695 of 1,696** transactions, the one
miss being $207 on a $5.0M endorsement that does not decompose — the source's own rollup exceeds the sum of
its own coverage rows, with no premium type to attribute it to. Guarded by
`assert_auto_coverage_premium_reconciles_to_lob`.

The vehicle × coverage rows carry a factor of type **2110 `Prem` / `premium_2`** which equals
`dim_premium.annual_premium` on **62,374 of the 62,379 issuing rows that carry it (99.99%)** — so the
rating engine's own calculated premium is recorded and agrees with the stored premium. This is what makes
coverage grain the right place to strike the auto rate: it is the engine's arithmetic, not a
reconstruction of it.

Premium by coverage, TPM Contractors auto:

| Coverage | rows | annual premium |
|---|---|---|
| **Liability** | 72,101 | **$340.01M** |
| Collision | 61,248 | $26.29M |
| Other Than Collision (comprehensive) | 63,458 | $8.43M |
| Personal Injury Protection | 46,722 | $6.08M |
| Uninsured Motorists | 53,764 | $4.50M |
| Medical Payments | 62,157 | $0.51M |
| Rental Reimbursement — COL / OTC | 25,176 | $0.93M |
| PIP — OBEL (`OBEL`) | 30,526 | $0.20M |
| Terrorism (`BLIA` / `BPHD`) | 1,269 | $0.24M |
| MRC — manually rated | 284 | $2.11M |
| Towing and Labor, PIP-Pedestrian, Liability-Primary, Underinsured | 18,323 | $1.05M |

Liability is **87% of auto premium**, so it is the coverage that matters for rate.

#### The formula (Liability)

```
per-vehicle premium = BASE               -- rate for this vehicle and this coverage
                    × Fac1 × Fac2        -- coverage-specific rating factors
                    × ILF                -- what they bought (limits)
                    × PRF                -- premium rating factor
                    × CDev               -- our filed deviation (ACTIVE on auto)
                    × RMF                -- what we decided (schedule rating)
                    × (1 + SRF)          -- signed adjustment, NOT a multiplier

policy auto premium = Σ over every vehicle × every coverage
```

There is **no `LCM` on auto** and **no exposure divisor** — `BASE` is already a rate per vehicle, not a
bureau loss cost per unit of exposure. The exposure is the vehicle count itself.

**Step 1 — every vehicle is priced separately, for every coverage it buys.** A single vehicle generates
a Liability row, and then a Collision row, a comprehensive row, a Medical Payments row and so on. Each
is an independent calculation with its own base rate and its own equation. This is why auto has ~30
factor types where GL has 15: the factor set is the union across coverages, not one chain.

**Step 2 — the base rate already includes the load.** Unlike GL, there is no bureau loss cost and no
`LCM`. `BASE` on auto is a chargeable rate for that vehicle and coverage.

**Step 3 — coverage-specific factors apply.** `Fac1`, `Fac2` and `PRF` carry the rating detail for the
particular coverage. The `eqn_element_N` field names are a direct signal that the engine composes a
**different equation per coverage**, so a formula verified on Liability should not be assumed to hold on
Collision.

**Step 4 — limits, then judgement.** `ILF` for the limit purchased, `CDev` for our filed deviation
(active here, unlike GL), `RMF` for schedule rating, and `SRF` as a signed adjustment applied as
`(1 + SRF)`.

**Step 5 — sum.** Policy auto premium is the sum over every vehicle and every coverage. The vehicle
count is the exposure, so the published rate is premium per vehicle.

Verified on `Issue` transactions, Liability coverage only:

| Population | rows | median actual ÷ predicted | within 1% |
|---|---|---|---|
| No `SRF` factor present | 7,797 | 1.0000 | **89.6%** |
| `SRF` present, applied as `(1 + SRF)` | 4,763 | 1.0025 | 45.5% |

`SRF` is a **signed adjustment applied as `(1 + SRF)`**, not a multiplier — the same shape trap as GL's
`Ded`. Values are `−0.10`, `−0.05`, `+0.20`. Applying it as `(1 − SRF)` gives a median of 0.9071 and
**zero** exact matches, so the sign convention is settled.

The `SRF`-bearing subset is **not fully reconstructed** — see open items. Treat auto Liability as
substantially reconstructed (~62% exact overall) rather than closed.

**This does not hold the published rate back, and that is the important structural point about auto.**
The per-vehicle formula above is the engine's *derivation* of a number the engine also *records* — the
`Prem` factor. So the rate does not depend on reproducing the chain: it sums what was calculated. A
reconstruction gap therefore limits rate-change **normalisation**, which needs the factors decomposed, and
not rate or exposure reporting, which need only the premium and the vehicle count.

#### Summary — what is and is not available for auto

| Measure | Available | Basis |
|---|---|---|
| Charged rate (premium per vehicle) | **Yes** | Σ vehicle × coverage annual premium ÷ live vehicle count |
| Rating premium at coverage grain | **Yes** | type 2399, reconciled to the line total |
| The engine's own calculated premium | **Yes** | `Prem` (2110), agrees with stored premium at 99.99% |
| Exposure (fleet size) | **Yes** | live `Covered Auto` risk count per transaction |
| Rate divisor from source | Not needed | a count basis takes divisor 1; there is nothing to scale |
| Per-vehicle rate | **Yes, in principle** | every vehicle's premium is recorded individually; not modelled, as no consumer needs it yet |
| Per-coverage rate split (Liability vs physical damage) | **Yes, in principle** | coverage type sits on each rated row; not modelled |
| Filed / manual rate benchmark | **No** | `BASE` on auto is already a chargeable rate, and there is no `LCM` — so there is no manual premium to deviate from, unlike GL and senior living |
| Loss-cost vs LCM split of rate change | **No** | neither exists on this line |
| Factor decomposition for normalisation | Partial | Liability substantially reconstructed, other coverages not — see open items |

**Two limitations of the published rate, both in the denominator rather than the numerator.**

**1. The fleet includes vehicles the engine did not price.** The exposure counts live `Covered Auto`
risks; the rating rows price a subset of them. Org-wide the difference is small — about 2% of vehicles —
and it runs one way only: no priced vehicle is ever missing from the count, so the rate is diluted rather
than overstated. It is concentrated rather than spread: **Sports and Entertainment carries about 4.5% of
its vehicles unpriced against roughly 0.3% in the other programs**, and a couple of large fleets account
for most of it. Whether a scheduled vehicle carrying no premium is a rating exposure is an underwriting
question, not a modelling one, so the count is left as the schedule records it.

**2. A cancellation publishes no rate, and auto is now the only line where that is true.** Cancelling an
auto policy deletes the whole vehicle schedule, so the exposure is zero and the rate resolves to no
exposure available. Note the numerator is *not* the obstacle — the coverage rows survive the cancellation
and carry a real annual price, several million dollars of it. Only the denominator vanishes. Every other
line publishes a cancellation rate.

#### Auto factor dictionary

Auto has its own factor set, which is why a rating-factor fact must be long/tidy rather than wide. Names
from `datas_ource_field`; note that many are exposed as `eqn_element_N`, i.e. the rating engine composes
a **different equation per coverage**.

| Factor | id | field | Definition / observed |
|---|---|---|---|
| **BASE** | 1666 | `eqn_element_5` | Base rate for the vehicle and coverage. Median 110, range 0.78–46,811. |
| **Prem** | 2110 | `premium_2` | The engine's calculated premium for the row. Equals `annual_premium` at 100%. |
| **VehType** | 2148 | `vehtype` | Vehicle type. Only 4 distinct values, median 1.0, min 0.10. |
| **Fac1 / Fac2** | 2032 / 2033 | `eqn_element_5` | Coverage-specific rating factors. `Fac1` ∈ {0.15 … 1.10}, `Fac2` ∈ {1.0, 1.04}. |
| **PRF** | 2112 | `eqn_element_3` | Premium rating factor — 27 values, 0.10–3.05. |
| **CDev** | 1996 | `eqn_element_6` | Company deviation. **Active on auto** (0.96–5.00), unlike GL where it is inert. |
| **RMF** | 411 | `eqn_element_8` | Rate modification factor — schedule rating. 155 distinct values, 0.44–5.593. |
| **ILF** | 2047 | `eqn_element_4` | Increased limit factor, 1.00–3.66. |
| **SRF** | 2128 | `eqn_element_4` | Signed adjustment applied as `(1 + SRF)`. Values −0.10, −0.05, +0.20. |
| **Adj** | 1972 | `eqn_element_1` | An **amount**, not a factor (−601 to +123). Do not multiply. |
| **DedFac** | 2015 | `eqn_element_7` | Deductible factor, 17 values. |
| **Ded** | 2014 | `eqn_element_11` | Constant 1.0 on auto. |
| **Adc1** | 1966 | `eqn_element_4` | An **amount** (1.0–41.8). |
| **Amount / Days** | 1980 / 2013 | `eqn_element_1/2` | Rental reimbursement: dollars per day (15/50/100) and number of days (30). |
| **TNC FAC** | 7309 | `tnc_factor` | Transportation network company. Constant 1.0. |
| **MedElim, WorkLoss, SPLia, RegFac, Lights, ESLtd, VWbuy, REPCOST, AccPrev, Mat Driver, Fleet, EWL FAC** | various | `eqn_element_*` | Coverage-specific elements, all at or near 1.0 in this book. |

**Traps.** `Adj` (1972) and `Adc1` (1966) are dollar amounts sitting in the factor table alongside
multipliers — multiplying by them corrupts the result. `1666 BASE` on auto is a **rate**, not a bureau
loss cost as it is on GL, and there is **no `LCM`** on auto at all.

### 4.5 KBSI Senior Living HPL

The subsidiary's primary line. Rated on a **weighted bed count**, and the only program whose full
rating chain — exposure, base rate, base premium, factor stack, unadjusted premium — is recorded
end to end in `FACTOR`.

**It is invisible to an exposure search built on GL.** KBSI HPL has **zero** `PREMIUMLIMIT` links,
therefore zero `LIMIT` type 549 exposure rows, and zero `9554` basis labels. The exposure is in
`FACTOR`.

#### Premium decomposition

**These amounts are WRITTEN premium.** Do not reconcile them against annual figures — the same components
on an annual basis are $18.5M rated and $1.66M terrorism, because PDS annualises this program at coverage
grain.

| Premium type | id | Amount | Role |
|---|---|---|---|
| `HEAPRO` | 14437 | **$375.2M** | Healthcare Professional Liability — the rated coverage |
| `HPLMRC` | 14415 | $14.6M | HPL **Manually Rated Coverage** — judgement priced, no exposure rate |
| `TERPER` | 14411 | $1.5M | Terrorism, percent — ancillary |
| `MP` | 14443 | $0 | Minimum premium |
| `TERROR` | 14410 | $0 | Terrorism premium |
| | | **$391.3M** | = policy premium, exactly |

None of these five types appear in the rating enumerations derived from GL. Exactly **one `HEAPRO` row
per location**, so the grain is clean and no MAX-vs-SUM decision arises.

#### Exposure — skilled bed equivalents

Beds are counted by level of care and converted to a common unit. **The conversion weights are stored
as named factors**, and they reproduce the stored `SBE` exactly:

| Care level | Factor | id | Weight |
|---|---|---|---|
| Skilled nursing | `SNF FAC` | 14076 | **1.00** |
| Intermediate / Alzheimer's | `INT/ALZ FAC` | 14075 | **0.80** |
| Assisted living | `ALF FAC` | 14078 | **0.60** |
| Independent living | `IL FAC` | 14077 | **0.35** |
| Adult day care | `ADC FAC` | 14067 | **0.35** |
| Sub-acute | `SUBACU FAC` | 14068 | 15–57 — a **rate**, not a weight; only 32 rows |

Verified against `dim_risk` occupied bed counts on single-care-level locations: skilled nursing 1.00
(n=3,985), intermediate/Alzheimer's 0.80 (n=116), assisted living 0.60 (n=659), independent living 0.35
(n=30) — exact, not approximate.

The resulting count is stored as **`SBE RATE` (14080)**, whose `datas_ource_field` is **`subtotal_0`** —
confirming it is a computed *count* despite the misleading type name. Raw bed counts sit in
`dim_risk` (`snf_occupied`, `alf_occupied`, `intalz_occupied`, `ilf_occupied`, `sub_acute_occupied`,
plus `*_licensed` equivalents), so `SBE` is independently recomputable.

Coverage: **720 of 721 HPL transactions**, **$390.4M of $391.3M** premium.

#### The formula

```
SBE        = Σ( beds at each care level × that level's weight )
                                         -- five kinds of bed, one common unit

BASE PREM  = base_rate                   -- price per skilled bed equivalent
           × SBE                         -- how much risk
           × TERR_FAC                    -- where it is (always a credit)

UNADJ PREM = BASE PREM                   -- the manual premium
           × CM                          -- claims-made basis
           × COR_DED                     -- corridor deductible
           × Ded                         -- retention (MULTIPLICATIVE here)
           × ILF                         -- limits (a DECREMENT here, 0.369-1.0)

charged     = UNADJ PREM
premium     × schedule_mod               -- underwriter judgement; VALUE NOT STORED
```

#### The schedule mod — recorded as a flag, not a value (upstream gap)

The step between the manual premium and the charged premium is the **schedule modification**: the
subjective underwriting judgement entered on the front end. PDS records **whether** one was applied and
**not what it was**.

`LOBRISKEXTENDEDDATA` type **14464 `Schedule mod`** is present on all 5,540 rated rows and holds only
**two values, `0` and `1`**. It correlates with the unexplained load perfectly, with no exceptions in
either direction:

| `Schedule mod` | rows | load is exactly 1.0000 | load ≠ 1.0000 | observed range |
|---|---|---|---|---|
| `0` | 48 | **48** | 0 | 1.0000 flat |
| `1` | 2,265 | 0 | **2,265** | 0.252 – 14.37 |

So the flag is trustworthy and the chain above is complete — the only thing missing is the modifier's
magnitude.

**Searched and ruled out** (KBSI HPL, exhaustive): `FACTOR` via `PREMIUMFACTOR` (21 types); `FACTOR` via
`LOBFACTOR` (only Corridor Retention and PL Retention factors); `FACTOR` via `TRANSACTIONDETAIL`, which
exposes three types unreachable through any junction — `POL PREM`, `ADJ PREM`, `PREM PERCENT` — of which
`POL PREM` is simply the charged policy total (equal to charged premium on 99.4%);
`COVERAGEEXTENDEDDATA`; `RISKEXTENDEDDATA`; `LOBEXTENDEDDATA`; `LOBRISKEXTENDEDDATA`;
`PREMIUMEXTENDEDDATA`; `TRANSACTIONEXTENDEDDATA` (fees, taxes, binding conditions, notes only);
`STATEEXTENDEDDATA` and `SUBCOVERAGE*` (no KBSI rows at all); `CONDITIONALDATA` (has no value column, so
it cannot store one); `RISKAMOUNT`, `STATEAMOUNT`, `BUREAU`, `BUREAUFACTOR` (empty account-wide).

**Two of those were eliminated by NAME and are now eliminated by TEST** — which is the difference between
a search that is exhaustive and one that only reads like it:

- **`PREM PERCENT` is terrorism, not a load.** 105 rows, four values between 0.001 and 0.03, on the same
  105 transactions that carry `ADJ PREM`. It is the rate behind `TERPER`.
- **The two LOB-level retention factors do not explain the residual.** `Corridor Retention Factor` (149
  transactions) and `Professional Liability Retention Factor` (97) are read by no model, which made them
  the credible candidate for the unrecorded load. The derived modifier equals the factor, or its inverse,
  on **1 transaction out of 149**. Median modifier is **1.17** where a corridor factor is present against
  **1.50** where neither is — an association, but nothing explanatory.

**Conclusion: the schedule mod value is not persisted anywhere in PDS.** This is an upstream gap in a
vendor-customised line, not a modelling oversight.

**Deriving it.** Every other input is exact, so the modifier solves algebraically:

```
schedule_mod = charged premium ÷ UNADJ PREM
```

This is an exact solve rather than an estimate, but four things must travel with it:

1. **Label it derived.** It is not a source value and must never be presented as one.
2. **Grain is the location, not the policy.** On 112 of 254 multi-location policies the load varies
   between locations, and the flag itself sits at `risk_lob` (per location per LOB) — so per-location is
   the correct grain. Compute the policy-level figure as `Σ charged ÷ Σ UNADJ`, not as an average of
   location ratios.
3. **Guard it with the flag.** Where `Schedule mod = 0` the derived value must be exactly 1.0000 — true
   on 48 of 48 today. That is a free and strong assertion: if it ever fails, either the chain is wrong
   or the source has changed.
4. **It absorbs anything unmodelled.** The derived value is a residual, so a new vendor factor would
   land inside it silently. The distribution supports treating it as suspect at the tails: median
   **1.328**, p25 0.92, p75 1.82, p99 4.62, and a maximum of **14.37** — with 21 locations above 5×,
   which is not a credible schedule modification. Carry a plausibility flag rather than
   trusting the value blindly, and investigate the tail as likely minimum-premium or manual-premium
   overrides rather than judgement. All figures are location grain on `Issue`, n = 2,313; restricting to
   the modified locations only (flag = 1) gives median **1.365**.

**Step 1 — count the beds, by level of care.** A facility is a schedule of beds at up to five care
levels: skilled nursing, intermediate/Alzheimer's, assisted living, independent living, adult day care.
Counts are held both **occupied** and **licensed**; rating uses occupied.

**Step 2 — convert them into one common unit.** A skilled nursing bed carries far more liability than an
independent living unit, so beds are weighted to a *skilled bed equivalent* and summed. This is the
mechanism the other programs lack — where Habitational refuses a rate on mixed bases and falls back to a
dominant one, KBSI converts every kind into one number with no residual.

**Step 3 — price per bed equivalent, then adjust for where it is.** `base_rate` is a chargeable rate per
SBE ($330–$4,000), not a bureau loss cost — there is no `LCM` on this program. `TERR_FAC` then applies
the territory, and on this book it is **always a credit** (0.2778–1.00). The result is `BASE PREM`.

**Step 4 — adjust for the coverage terms.** Claims-made basis, corridor deductible, retention and limit.
Note both reversals against GL: `Ded` is **multiplicative** here rather than a credit fraction, and
`ILF` is a **decrement** (0.369–1.00) rather than a load. The result is `UNADJ PREM`, the manual premium.

**Step 5 — a load is applied that is not recorded.** Charged premium sits at a median 1.328× the
unadjusted premium. The systematic part behaves like the loss-cost multiplier that GL carries explicitly;
the dispersion around it (p25 0.92, p75 1.82) is negotiation and judgement.

**Where the premium is booked is not where you would guess.** PDS keeps the rated premium on one set of
location records and the ancillary premium on a *separate* set: terrorism sits on the location records that
carry **no** rated premium, and no location ever carries both rated and manually rated premium. The
`Rating Basis` / `Schedule mod` flag pair is present on exactly the rated set and absent from the other, so
the flag is a perfect discriminator between the two — which is what
`assert_kbsi_rated_and_manual_premium_are_exclusive` asserts.

Verified on `Issue` transactions, n = 2,313:

| Step | Result |
|---|---|
| `BASE PREM = base_rate × SBE × TERR_FAC` | **100.0% exact** |
| `UNADJ PREM = BASE PREM × CM × COR_DED × Ded × ILF` | **100.0% exact** |
| `charged premium ÷ UNADJ PREM` | median **1.328**, p25 0.92, p75 1.82 |

The two recorded steps are exact — the earlier apparent failure of `BASE PREM = base_rate × SBE` was
caused by omitting the **territory factor**, which is the whole discrepancy.

#### KBSI factor dictionary

| Factor | id | field | Definition / observed |
|---|---|---|---|
| **BASE** | 1666 | `base_rate_0` | Base rate **per skilled bed equivalent**. 27 values, $330–$4,000. A rate, not a loss cost — there is no `LCM` on this program. |
| **SBE RATE** | 14080 | `subtotal_0` | The weighted bed count (a count, despite the name). 5.4–533. |
| **TERR FAC** | 7129 | `eqn_element_12` | **Territory factor.** 24 values, 0.2778–1.00 — always a credit. Applies before base premium. |
| **BASE PREM** | 1673 | `subtotal_2` | `base_rate × SBE × TERR_FAC`. Also duplicated as `PREM BASE` (9672) and `SubPrem` (2132) on the same `subtotal_2/3` values. |
| **CM** | 9222 | `eqn_element_21` | Claims-made factor. 3 values, 0.60–1.00. |
| **COR DED** | 14072 | `eqn_element_20` | **Corridor deductible** factor. 0.70–1.00. |
| **Ded** | 2014 | `eqn_element_19` | Deductible / retention factor — **multiplicative here**, 13 values, 0.20–1.00. Contrast GL, where `Ded` is a credit fraction. |
| **ILF** | 2047 | `ilf` | Increased limit factor, 0.369–1.00 — a **decrement** on this line, unlike GL. |
| **UNADJ PREM** | 1932 | `subtotal_4` | The manual premium after the factor stack. |
| **HC / HHC / RC Rate** | 14440 / 14439 / 14438 | `eqn_element_*` | Home care, home health care, residential care rates. Constant 1.0 — not in use. |
| **SRV PREM** | 14085 | `subtotal_1` | Service premium. 3 rows, $20. |

**KBSI supports a deviation-off-manual measure, and Contractors does not.** The distinction is that
KBSI's `UNADJ PREM` is a genuine manual premium, reproducible from its own inputs at 100%, whereas
Contractors' stored composite rate is a derivation artefact. The KBSI deviation is the charged premium
over the unadjusted premium.

Two cautions. The median load of **1.328** is systematic and is **not recorded anywhere as a factor** —
it behaves like a loss-cost multiplier applied outside the factor set, and it should be identified
before any deviation *level* is published. And bed counts exist in both **occupied** and **licensed**
form; `SBE` is built on **occupied** (median weighted 85 against 115 licensed) — which is not an
assumption but a value PDS records, on `Rating Basis` (14463), reading `Occupied` on every rated location.

---

### 4.6 Umbrella / Excess — priced on a stored equation, not on an exposure

This is the one line with **no exposure of its own** anywhere in PDS: no class codes, no exposure limits,
no risk schedule. The published measure is therefore a *relativity* — the excess premium as a percentage
of the underlying primary premium — rather than a rate per unit.

#### Premium decomposition

Umbrella premium decomposes by **attachment layer**, not by coverage: one premium row per layer of limit.

| Component | Types | Note |
|---|---|---|
| Attachment layers | `UMBLIM`, `1MLIM` … `5MLIM` | The rating premium. `UMBLIM` is the first layer and the only one carrying a factor set. |
| Manually rated coverage | `MRC` | The negotiated plug. Frequently **negative** on this line. |
| Terrorism / state | `TERROR`, `MP`, `UMSTAT` | `UMSTAT` sums to zero everywhere it appears. |

`layers + MRC + terrorism/state = policy premium` holds to the cent, asserted by
`assert_umbrella_premium_identity`.

#### The formula — the first layer reproduces exactly

The engine stores a free-form rating equation on the `UMBLIM` premium row, and its product reproduces the
charged layer premium:

```
eqn_element_0  ×  eqn_element_1  ×  ilf   =   UMBLIM premium

    100,000    ×      13.593     ×  0.22  =     299,046
  1,450,000    ×       1.058     ×  0.22  =     337,502
  4,708,440    ×       1.316     ×  0.22  =   1,363,188
```

Verified on the issuing population with a median absolute difference of **$0.12** — rounding. `rmf` is
present and is 1.000 on all but a handful of rows.

**It reproduces `annual` premium, not written.** On cancellations the written form fails on essentially
every row while the annual form holds, which is independent confirmation from the factor store that annual
premium is the whole-term price a coverage was struck at and written is what gets restated on
cancellation.

#### Why none of it is readable as a rate

- **`eqn_element_0` and `eqn_element_1` are positional equation slots, not typed fields.** `eqn_element_0`
  ranges from about 58,000 to 221.9M and `eqn_element_1` from 0.002 to 13.593, moving inversely — one row
  carries a basis of 100,000 with a factor of 13.593 and the next 4.7M with 1.316. Whoever rated each
  policy filled the two slots differently: sometimes limit × rate, sometimes premium × relativity. Only
  the *product* means anything, and it has no stated unit. `factor_description` is empty throughout, so
  field names come from `datas_ource_field`.
- **The equation's basis is not the underlying primary premium.** `eqn_element_0 × eqn_element_1` runs a
  median **2.19×** the matched primary GL premium on Contractors and **1.64×** on Habitational, and equals
  it on almost no transactions. So the engine's `ilf` and the published relativity are not comparable
  measures, and the two only look close because the engine's basis is about twice the denominator the
  relativity uses.
- **`ilf` is the only genuine source rate, and only for the first layer.** 7 distinct values
  (0.16 / 0.19 / 0.22), a **decrement** on this line unlike GL. The layers above the first carry no rate
  at all — only `rmf`, `subtotal_0` and `adjusted`, where `adjusted` is simply a stored copy of the layer
  premium.

So the relativity stays **derived**, and that is a decision rather than a gap: there is no source premium
to divide by and no source percentage to read.

#### The schedule of underlying insurance — no premium, but a usable key

The underlying insurance is carried as unrated coverages, one per underlying line
(`General Liability Information` and its automobile / employers / other siblings). They hold **no premium
and no coverage value**. `Umbrella Rating Stats` is likewise empty — a $0.00 premium row with no factors.
There is no recorded underlying premium anywhere, which is why the denominator is derived by matching.

What the schedule does record, in coverage extended data, is the underlying **policy number** (7458), its
term (7459 / 7460) and its **carrier** (7461). Two consequences:

- **The excess sits over our own primary, on source evidence.** The recorded carrier is one of our own
  companies on **1,229 of 1,241** transactions; only 12 name a third party. Excess written over another
  carrier is a rounding error in this book, and any measurement suggesting otherwise should be treated as
  a matching artefact first.
- **The recorded number cross-validates the matching.** Where it resolves to a policy in our book it
  agrees with the derived denominator exactly on the large majority of transactions, sharing no logic with
  the account matching. Where it disagreed, the matching had summed several concurrent primaries while the
  schedule named one, understating those relativities by about 2.5×.
  `int_pds_policy_underlying_match` now keeps only the named policy where the schedule names exactly one and
  that policy is already a candidate.

Roughly 40% of transactions have no usable recorded number — either none at all, or a quote-stage or
placeholder value such as `quote` / `tbd`, entered because the excess was bound before the primary had a
number. On those the matching rules remain the only path, so they cannot be retired.

---

## 5. Open items

| # | Item | Detail |
|---|---|---|
| 1 | **`REDUCT` (2118) — RESOLVED, it is an ordinary multiplier** | The original reading, that reconstruction was out by a factor of exactly 1/3 on 35 rows, was a **reconstruction that had left `REDUCT` out of the product**. Put in beside `RMF`, every one of the 42 issuing `334` rows where it is not 1.0 reconstructs at a median ratio of **exactly 1.0000**, at values from 0.70 to 22.64 — above and below one alike. Drop it back out and the ratio equals the factor, which is what the 1/3 was. Nothing to fix in any model: the rate is struck on charged premium, so this never reached a published number, and the chain in §3 already carried the factor. |
| 2 | **RSI premium leakage — RESOLVED, the premium is named, not missing** | The stated cause was that premium types outside the rating enumeration were not captured. They are now all named, and the written-premium identity closes at **$0.00 on all 2,029 rated RSI transactions**. 520 transactions still sit below 0.90× policy premium, and every dollar of the gap is accounted for by components that are outside a rate for a stated reason — `@P` coverage-builder endorsements, flat charge, and the manually rated plug. Rate-level leakage, meaning rateable premium behind a zero exposure, is **$48.1K and 100% attributed**. See §4.2. |
| 3 | **RSI `336` dollar rounding — RESOLVED as guidance; no model or test relies on the wrong side of it** | The rule stands and is worth keeping: any tolerance on products/completed-ops rows must be absolute, because the premiums are single-digit dollars. Checked at close-out — **no test in the project measures a `336` row on a relative tolerance**. Note the same problem inverts at the rate grain: on a coverage carrying a fraction of one exposure unit, a sub-dollar premium rounding becomes a multi-dollar *rate* gap (78 RSI coverages, median exposure 0.20 units, median premium $15). So a rate test needs the absolute tolerance where premium is small **and** a relative one where exposure is large; neither alone is safe. |
| 4 | **Composite rate quality — RESOLVED, by not using the field** | The 236 distinct values of type 14399 include `0.0000000` and values up to `5.7e12`, which cannot be rates. No validity filter is needed, because **no model reads 14399 at all**: the stored value is a derivation artefact that scales with coverage count (see §4.3), so the published Contractors rate is derived as `CMPRAT`/`CMGP*` premium ÷ exposure × 1000 instead. The filter this item asked for would have been work spent making an unusable field safe to use. If anything ever does need 14399, restrict it to single-class-code policies and filter the range — but prefer the derived rate. |
| 5 | **`CODEV` never exercised on GL** | Constant 1.0 in both GL programs. It **is** active on auto (0.96–5.00). A GL formula that ignores it silently breaks if it is ever used, so it is retained in the §3 formula. |
| 6 | **Auto `SRF` subset not reconstructed — DOES NOT BLOCK THE RATE; still open for normalisation** | Unchanged as a measurement: Liability closes at 89.6% without `SRF` and 45.5% with it, and something else varies on those 4,763 rows. What changed is what depends on it. **Nothing in the published auto rate does**, because the rate sums the engine's own recorded per-vehicle premium (`Prem`, 2110) rather than reproducing the equation that produced it — the same resolution shape as item 4, where the answer was to stop needing the field. So this is now scoped to rate-change **normalisation**, which does need the factors decomposed, and it is not on the critical path for rate or exposure reporting. |
| 7 | **Auto non-Liability coverages not reconstructed — same reclassification as item 6** | Only Liability (87% of auto premium) has been tested; Collision, comprehensive, PIP, UM, Medical Payments and the rental-reimbursement coverages each compose a **different equation**, visible in the `eqn_element_N` field names. **The rate is unaffected**: every coverage's premium is summed from what the engine calculated, whatever equation produced it, and that sum reconciles to the line-of-business total the source records independently — which is a completeness check across *all* coverages, not just the reconstructed one. Open for normalisation only. |
| 8 | **KBSI schedule mod value is not persisted — upstream gap** | `LOBRISKEXTENDEDDATA` type 14464 `Schedule mod` is a **boolean** (0/1) that correlates perfectly with the load but carries no magnitude. Exhaustively searched and ruled out across every PDS store, the last two candidates by numeric test rather than by name — `PREM PERCENT` is the terrorism rate, and the two unread LOB retention factors match the derived modifier on 1 transaction of 149 — see §4.5. **Search closed.** **Now derived in `int_pds_kbsi_schedule_mod`** as `charged ÷ UNADJ PREM`, labelled derived, guarded by the flag (`assert_kbsi_schedule_mod_is_one_when_flag_is_zero`) and tail-flagged. Still worth raising upstream with the vendor. **Published on cancellations and reinstatements** — an earlier version of this item said it was withheld there because the charged and manual premiums "do not restate together", which is false: a cancellation in PDS restates the annual premium, the manual premium and the bed schedule and rewrites only the written premium, so the modifier carries forward from the prior transaction unchanged on all 29 such transactions. |
| 9 | **KBSI occupied vs licensed beds — RESOLVED** | PDS records which basis the rating used, on `LOBRISKEXTENDEDDATA` type **14463 `Rating Basis`**, one row per location. It reads `Occupied` on every rated location in the book. The models read the flag rather than assuming, and `assert_kbsi_rating_basis_is_occupied` fails the build on any other value — licensed beds run materially higher and there is no example of a licensed-rated policy to build a path from. |
| 10 | **Amounts stored as factors on auto — no live exposure; a standing caution for normalisation** | `Adj` (1972, −601 to +123) and `Adc1` (1966, 1.0–41.8) are dollar amounts sitting in `FACTOR` beside multipliers, so any generic "multiply all factors" logic corrupts auto. Checked at close-out: **no model multiplies auto factors at all** — the only auto factor any measurement reads is `Prem` (2110), and it is read as a premium, which is what it is. So the trap is unsprung, and it stays listed because it is exactly the kind of thing a factor-normalisation model would walk into. |
| 11 | **The derived term factor is computed from a mismatched pair of dates** | The annualisation applied to rollup-grain premium is 365 ÷ term days, where the term is `policy_inception_date` (stabilised to one value per policy) minus `policy_expiration_date` (deliberately **not** stabilised). On a transaction whose booked inception was later amended, the two dates therefore come from *different transactions* and the term is one that never existed — currently 382 days on a policy written for 365, which also lands outside the 365/366 no-scaling band and so gets scaled when it should not be. Small population, listed by `warn_policy_effective_date_amended_by_later_transaction`. **No longer reaches any rate numerator**: auto was the last numerator taken at rollup grain and now reads coverage grain, which the source annualises itself. The factor survives only on `policy_premium_annual`, the denominator of an excess relativity, which has no coverage-grain equivalent. Fixing it means either stabilising expiration — which the project deliberately forbids, because a term can genuinely be extended — or deriving the term from the booked inception on the same transaction. **Not decided; the exposure is one measure on a handful of policies.** |

---

## 6. Not yet documented

| Program / line | Note |
|---|---|
| Umbrella / Excess | **Now documented — see §4.6.** |
| Legacy Velocity (pre-2025 cutover) | A different platform with one generic exposure column and no basis label, no bureau loss cost and no factor stack. It records its own rate (`comp_rate_`) but nothing to decompose. Rate is available; rating mechanics are not. |
