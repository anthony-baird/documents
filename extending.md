# Extending the foundation

Five recipes for changing these models without breaking the contract they publish. The foundation is built
so unfamiliar business either flows through correctly or refuses to publish and says why; it is never
configured to produce a number.

---

## 1. Stage a PDS table nobody has staged yet

Eight vendor tables are neither staged nor declared as sources: `BUREAU`, `STATEBUREAU`, `BUREAUFACTOR`,
`TAXSURCHARGEFACTOR`, `RISKPARTY`, `ACCOUNTPARTY`, `SUBCOVERAGE`, `SUBCOVERAGELIMIT` — the last empty at
source.

**Where new models go.** Every layer is organised by source: `1_stg/pds/`, `2_int/pds/`, `4_fact/pds/`, with
Velocity treated as part of the `pds` domain since it is the legacy half of the same policy system. Names
carry the source too — `stg_pds_*`, `int_pds_*`, `fact_pds_*`. The exception is `3_dim/conformed/`, which
holds dimensions that span every source and are therefore not source-scoped. Some pre-existing PDS models
still sit at their layer root with unprefixed names; new work follows the convention above.

Add the bare uppercase name to the `pds` source in `models/sources.yml` — no description, freshness or
`schema:`, since the dev/prod switch is a block scalar on the source itself. Then a flat `select`:

```sql
-- models/1_stg/pds/stg_pds_bureau.sql
select
    bureauid as bureau_id,
    _fivetran_synced
from {{ source('pds', 'BUREAU') }}
where not _fivetran_deleted
```

PDS columns are lowercase-concatenated (`bureauid`, `risklobid`, `policytransactionid`), so aliasing every
one is the whole job of the layer. `where not _fivetran_deleted` is that exact form, not `= false`.
Surrogate keys and casts are per-model rather than conventions. Declare the model in
`models/1_stg/pds/__pds_models.yml` with `unique`/`not_null` on the key and `relationships` where a foreign
key exists.

**The trap.** `policytransactionid` is the transaction's primary key; `transactionid` is a coarse storage
scope shared across unrelated policies and lines, so joining detail on it alone leaks rows silently — it
once inflated one policy's vehicle count from 444 to 12,611. Detail needs `storageid` **and**
`transactionid`, or the documented junction ([relational_model.md](relational_model.md#the-spine)).

---

## 2. A new exposure basis appeared at source

You meet this as a red test: `assert_all_exposure_bases_are_mapped`, naming the unmapped basis.

`macros/exposure_basis_type.sql` is a `case` with **no `else`**, so an unrecognised basis maps to NULL — no
comparison group, no published rate. A new unit therefore surfaces as a broken build rather than as an
uncomparable rate inside someone's median. Add it to that `case`, choosing one of five types; that choice is
the only judgement, and it sets comparability:

| If the new unit is | Basis type | Grouping |
|---|---|---|
| A dollar amount | `Currency` | Basis and divisor. Comparable across programs. |
| Square footage | `Area` | Same. |
| A count of business objects | `Count` | Basis, divisor **and program** — a "unit" differs per program. |
| A fixed charge with no unit | `Flat` | No rate is ever published. |
| A percentage of another premium | `Relativity` | By program. |

`rate_comparison_group` keys off the basis type, so grouping then follows automatically. Note the test reads
the **raw source population**, so a basis on coverages that never get rated still has to be mapped, and that
four bases in the macro are derived here rather than read from source.

**The mistake:** mapping a count to `Currency` so it compares across programs. Nothing downstream objects.

---

## 3. Add a program

**Usually there is nothing to add.** `models/2_int/pds/int_pds_policy_rating_method.sql` reads only what a policy
contains — premium types, coverage types, whether a usable exposure is present, whether it owns vehicles,
its line of business — with no subsidiary or program predicate. A new program normally resolves to one of
ten existing methods with no code change: `composite`, `multiple_composite`, `class_code`,
`bed_equivalent`, `vehicle_count`, `hired_non_owned_only`, `underlying_relativity`, `flat_charge`,
`negotiated_premium`, `no_exposure_available`. Those map onto the exposure strategies
`pds_coverage_exposure_limit`, `pds_factor_exposure`, `risk_count`, `pct_of_underlying`, `none`.

So build nothing first: run the models and see what the program resolved to and whether its premium identity
holds. `no_exposure_available` is a correct outcome, not a bug.

**The deliverable is a worked-example test**, following
`tests/assert_habitational_gl_worked_examples.sql` and its four siblings:

```sql
with expected as (
    select * from values
        ('CZ20NYGL0000-01', 'composite', 27818056, 167187, 6.010)
        as t (policy_number, rating_method, exposure_amount, rated_premium, rate)
),
actual as (
    select p.policy_number, f.rating_method, f.rated_premium, f.rate
    from {{ ref('fact_pds_policy_exposure_rate') }} f
    inner join {{ ref('dim_policy') }} p on f.policy_key = p.policy_key
    where f.transaction_type = 'Issue'
)
select ... from expected e
left join actual a on e.policy_number = a.policy_number
where a.policy_number is null                            -- a vanished policy must fail
   or a.rating_method is distinct from e.rating_method
   or a.rate is null                                     -- required before any abs()
   or abs(a.rate - e.rate) >= 0.0005                     -- source displays 3 dp
```

Expected values are inline `values`, not a seed, pinned to figures traceable on the source system's own
screens. The commented lines are the load-bearing ones: without the `is null` guard, `abs()` on a NULL
returns NULL and the test passes precisely when a rate stops resolving.

Beyond the test, only a divisor if the program quotes per-something, and a comparison-group entry if it
introduces a new basis.

**The mistake, with precedent.** There is exactly one program-scoped carve-out in the rating path — the
`is_back_solved_class_code` macro, expanding to `subsidiary = 'TPM' and program = 'Contractors' and
rating_method = 'class_code'`. It is a **data-quality correction** for policies where an underwriter
back-solved a composite price through the class-code rows, not a rating rule, and its header says to resist
adding a second. A second one means the mechanism is wrong, not that the program is special.

---

## 4. Add a column to a fact model

Document it in `models/4_fact/pds/__pds_fact_schema.yml` under its model, following the rate-layer
convention (`description:` plus tests) rather than the older SureMGA entries in the layer-root
`__fact_schema.yml`, which carry only `unique`/`not_null`.

- **If it is not additive** it belongs in the [Never sum, never
  average](fact_models.md#never-sum-never-average) table, and its description should say so. Rates, factors,
  cumulative snapshots, memo columns and anything premium-weighted are not.
- **The term fact publishes ONE rate, valued at `rate_as_of`.** Do not add a second valuation anchor beside
  it — a suffixed `_at_inception` rate, or anything equivalent. It splits the rate from the ladder that
  decomposes it, so a terms factor ends up dividing out something the published price never contained, and
  `assert_term_rate_is_unanchored` fails the build on it. If a second anchor is genuinely needed, that is a
  design change to argue for, not a column to add.
- **A sometimes-absent value needs a paired absence reason, never a plausible substitute.** A recorded
  neutral value is published as-is; one that does not exist is NULL with a reason.
- **A new gate, threshold or absence reason must be documented in
  [publishing_a_rate.md](publishing_a_rate.md), and a new reason string must be registered in
  `assert_absence_reasons_are_registered`.** The build fails on an unregistered string, which is the point:
  a threshold that lives only in the model is one no consumer can discover, and withholding a value while
  hiding the number that explains it is worse than not computing it. This rule exists because a gate that
  suppressed a normalized rate on more than a hundred transactions sat undocumented \u2014 every test passed,
  because the tests checked that an absent rung had *a* reason and never checked that anyone could look the
  reason up.
- **Publish the number behind a gate beside the value it gates**, so a consumer can widen it deliberately
  rather than discovering it by surprise \u2014 as `terms_coverage_share` sits beside the normalized rate.
- **On `fact_pds_premium_earned_monthly` and `fact_pds_policy_renewal_rate_change` a new column usually belongs
  upstream** — both publish no measures of their own by design, and a test recomputes them against their
  parent.

---

## 5. Read a failing test properly

Roughly seventy standalone assertions guard these models, plus five `warn_*` tests. Several encode rules
written down nowhere else in the repository, so invert the usual assumption:

> **A failing assertion here is more likely to be a real finding than a stale expectation.**

- **Tolerances sit at the precision the source displays.** Widening one turns a detected error into an
  undetected one.
- **A pinned policy that no longer resolves is a failure, not an obsolete fixture.**
- **One policy failing usually means the source re-stated it; all of them failing means the model changed.**

Run tests with **`dbt build --no-partial-parse`**. A stale parse cache silently runs a subset of the suite
and a green build then means nothing — a *falling* "Found N data tests" count is a cache problem rather than
fewer tests. Then confirm the edit landed by re-reading the changed lines; a green build will not catch a
valid column swapped for another valid column.
