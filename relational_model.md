# The raw PDS relational model (Policy Data Store, from Insurity)

**This file is about the raw vendor tables, not the reporting models.** It applies when writing SQL against
`stg_pds_*` — building a new staging model, adding an intermediate model, or investigating a number at
source. Consuming the published facts requires none of it; [start_here.md](start_here.md) covers that.

The point of the file is that PDS is a large normalised vendor schema with several keys that look
interchangeable and are not. Joining it the way it looks like it should join produces plausible wrong
numbers rather than errors. What follows is the model Insurity actually intends.

PDS is the primary policy source (Azure SQL, landed in Snowflake by Fivetran as `raw.qsazuredb_pds` — or
`raw.qs_azuredb_pds` on the `dev_testing` target).

Two sections carry most of the risk: [the two transaction keys](#the-spine) and [the premium
junctions](#premium-how-amounts-attach). Between them they account for more wrong numbers than everything
else here combined.

Insurity documents PDS as **three traversal patterns** off the same policy/transaction spine. Every reporting question resolves to one of them:

| Pattern | Question it answers | Grain |
|---|---|---|
| **Detail By Risk** | premium / limits / deductibles / factors for a specific rated risk (location, vehicle, class) | risk × LOB × coverage |
| **Detail By LOB** | premium, taxes & surcharges, bureau and LOB factors rolled to line of business and state | LOB or state |
| **Detail By Transaction** | who is on the policy, billing, cancellation, commission, transaction-level premium | policy transaction |

## The spine

```
POLICYHEADER ──1:N── TRANSACTION ──┬── STATE ── RISK
   (policy)          (endorsement/  ├── LOB
                      version)      ├── ACCOUNT / CANCELLATION / BILLINGINFORMATION
                                    ├── TRANSACTIONPARTY
                                    └── TRANSACTIONAMOUNT
```

- `POLICYHEADER.policyid` → `TRANSACTION.policyid`. `POLICYHEADER.currenttransactionid` points at the in-force version.
- **`TRANSACTION` has two keys and they are not interchangeable:**
  - `policytransactionid` — the transaction's own PK. Transaction-scoped children (`LOB`, `STATE`, `ACCOUNT`, `CANCELLATION`, `BILLINGINFORMATION`, `TRANSACTIONAMOUNT`, `TRANSACTIONPARTY`) join on this.
  - `transactionid` — the storage/version scope stamped on nearly every detail row (`RISK`, `COVERAGE`, `PREMIUM`, `LIMIT`, `DEDUCTIBLE`, `FACTOR`, `ADDRESS`, `VEHICLE`, `INSURANCEPARTY`, …). Use it to keep detail rows inside the correct policy version, never as a join to `POLICYHEADER`.
- `RISK` hangs off **`STATE`**, not directly off `TRANSACTION`: `STATE.stateid` → `RISK.stateid`.
- Risk and LOB are many-to-many, resolved by the **`LOBRISK`** junction (`risklobid` = `riskid` × `lobid`). `LOBRISK.risklobid` — not `riskid` — is the key that coverage and risk-level limits/deductibles hang from.

## Pattern 1 — Detail By Risk

```
POLICYHEADER → TRANSACTION → STATE → RISK
                          → LOB
                    RISK × LOB = LOBRISK (risklobid)

RISK ──┬── LOCATION ── LOCATIONADDRESS ── ADDRESS
       ├── VEHICLE
       └── RISKAMOUNT ── PREMIUMAMOUNT

LOBRISK ──┬── LOBRISKLIMIT ────── LIMIT
          ├── LOBRISKDEDUCTIBLE ─ DEDUCTIBLE
          └── COVERAGE ── PREMIUM ──┬── PREMIUMLIMIT ──────── LIMIT
                                    ├── PREMIUMDEDUCTIBLE ─── DEDUCTIBLE
                                    ├── PREMIUMFACTOR ─────── FACTOR
                                    └── PREMIUMDETAILAMOUNT ─ PREMIUMAMOUNT
```

Canonical join (Insurity's own sample, expressed against our staging models):

```sql
from      stg_pds_policy_header       p
join      stg_pds_transaction         t   on t.policy_id           = p.policy_id
join      stg_pds_state               st  on st.policy_transaction_id = t.policy_transaction_id
join      stg_pds_lob                 lob on lob.policy_transaction_id = t.policy_transaction_id
join      stg_pds_risk                r   on r.state_id           = st.state_id
join      stg_pds_lob_risk            lr  on lr.risk_id = r.risk_id and lr.lob_id = lob.lob_id
join      stg_pds_coverage            c   on c.risk_lob_id        = lr.risk_lob_id
join      stg_pds_premium             pr  on pr.coverage_id       = c.coverage_id
join      stg_pds_premium_detail_amount pda on pda.premium_id     = pr.premium_id
join      stg_pds_premium_amount      pa  on pa.premium_amount_id = pda.premium_amount_id
```

Risk-level attributes are one-per-risk by design: `LOCATION`, `VEHICLE`, and `RISKAMOUNT` each carry `riskid`, and `LOCATIONADDRESS` resolves a location to exactly one `ADDRESS`.

## Pattern 2 — Detail By LOB

```
POLICYHEADER → TRANSACTION ──┬── LOB
                             └── STATE

STATE ──┬── STATEBUREAU ── BUREAU
        │        └── BUREAUFACTOR ── FACTOR
        ├── STATEAMOUNT ── PREMIUMAMOUNT
        └── TAXSURCHARGE ──┬── TAXSURCHARGEAMOUNT ── PREMIUMAMOUNT
                           └── TAXSURCHARGEFACTOR ── FACTOR

LOB ──┬── LOBLIMIT ────── LIMIT
      ├── LOBDEDUCTIBLE ─ DEDUCTIBLE
      ├── LOBFACTOR ───── FACTOR
      └── LOBAMOUNT ───── PREMIUMAMOUNT
```

`TAXSURCHARGE` belongs to `STATE` (`stateid`), not to `LOB` — state taxes and surcharges cannot be attributed to a line of business through this model. `TAXSURCHARGE.typeid` resolves against `TYPE` for the surcharge label (`stg_pds_tax_surcharge` already joins it).

## Pattern 3 — Detail By Transaction

```
POLICYHEADER → TRANSACTION ──┬── ACCOUNT ── ACCOUNTPARTY ─────┐
                             ├── CANCELLATION                 │
                             ├── BILLINGINFORMATION           │
                             ├── TRANSACTIONAMOUNT ── PREMIUMAMOUNT
                             ├── TRANSACTIONPARTY ────────────┤
                             ├── STATE                        │
                             └── RISK ── RISKPARTY ───────────┤
                                                              ▼
                                                       INSURANCEPARTY
                                                         ├── INSURANCEPARTYADDRESS ── ADDRESS
                                                         └── COMMISSION
```

`INSURANCEPARTY` is the single party master. The same party is reached three different ways depending on the role being asked about — insured/agent on the transaction (`TRANSACTIONPARTY`), account-level party (`ACCOUNTPARTY`), or party attached to a specific risk (`RISKPARTY`). `COMMISSION.percent` is a text column at source and must be cleaned before casting (`stg_pds_commission` does this).

## Premium: how amounts attach

`PREMIUMAMOUNT` is the one amount table for the whole model, and it is reached through a different junction depending on the grain you want. Every one of these junctions is a separate row per amount — **never join two of them in the same query** or premium will multiply:

| Grain | Junction | Staging model |
|---|---|---|
| Coverage / rated line | `PREMIUMDETAILAMOUNT` | `stg_pds_premium_detail_amount` |
| Risk | `RISKAMOUNT` | `stg_pds_risk_amount` |
| LOB | `LOBAMOUNT` | `stg_pds_lob_amount` |
| State | `STATEAMOUNT` | `stg_pds_state_amount` |
| Tax / surcharge | `TAXSURCHARGEAMOUNT` | `stg_pds_tax_surcharge_amount` |
| Transaction total | `TRANSACTIONAMOUNT` | `stg_pds_transaction_amount` |

`stg_pds_premium_amount` exposes `written_premium`, `annual_premium`, and `incremental_premium` (which zeroes out `statustypeid = 1474`, "Previously Deleted"). `premiumamounttypeid` resolves against `TYPE`.

## Cardinality assumptions Insurity documents

The vendor diagram calls these out explicitly as uniqueness requirements. They are **assumptions the model relies on, not enforced constraints** — if any is violated in the landed data, joins fan out and premium double-counts. Worth asserting with dbt tests where a report depends on it.

Detail By Risk:
1. `RISKID` in `VEHICLE` must be unique
2. `RISKID` in `LOCATION` must be unique
3. `LOCATIONID` in `LOCATIONADDRESS` must be unique — i.e. a location cannot have multiple addresses
4. `PREMIUMAMOUNTID` in `RISKAMOUNT` must be unique
5. `RISKID` in `LOBRISK` must be unique

Detail By LOB:
1. `STATEID` in `STATEBUREAU` must be unique
2. `FACTORID` in `TAXSURCHARGEFACTOR` must be unique
3. `TAXSURCHARGEID` in `TAXSURCHARGEAMOUNT` must be unique
4. `PREMIUMAMOUNTID` in `TAXSURCHARGEAMOUNT` must be unique

Detail By Transaction:
1. `PREMIUMAMOUNTID` in `TRANSACTIONAMOUNT` must be unique
2. `INSURANCEPARTYID` in `RISKPARTY` must be unique
3. `INSURANCEPARTYID` and `ADDRESSID` in `INSURANCEPARTYADDRESS` must be unique
4. `INSURANCEPARTYID` in `ACCOUNTPARTY` must be unique

## Junction (bridge) tables

Highlighted in red on the vendor diagram — these hold no business attributes, only a pair of foreign keys, and each one is a fan-out risk:

`LOBRISK`, `LOBRISKLIMIT`, `LOBRISKDEDUCTIBLE`, `LOBLIMIT`, `LOBDEDUCTIBLE`, `LOBFACTOR`, `LOBAMOUNT`, `PREMIUMLIMIT`, `PREMIUMDEDUCTIBLE`, `PREMIUMFACTOR`, `PREMIUMDETAILAMOUNT`, `RISKAMOUNT`, `STATEAMOUNT`, `STATEBUREAU`, `BUREAUFACTOR`, `TAXSURCHARGEAMOUNT`, `TAXSURCHARGEFACTOR`, `TRANSACTIONAMOUNT`, `TRANSACTIONPARTY`, `RISKPARTY`, `ACCOUNTPARTY`, `LOCATIONADDRESS`, `INSURANCEPARTYADDRESS`.

## `TYPE` — the universal code table

Every `*typeid` column in PDS (`transactiontypeid`, `lobtypeid`, `risktypeid`, `coveragetypeid`, `premiumtypeid`, `premiumamounttypeid`, `limittypeid`, `deductibletypeid`, `factortypeid`, `statustypeid`, `cancellationtypeid`, `billingplantypeid`, `paymentplantypeid`, `partytypeid`, `basistypeid`, …) resolves against `TYPE.typeid` for its label (`stg_pds_type` → `type_enum`, `type_description`, `category_id`). There is no per-domain lookup table.

## Limits: the documented junctions don't cover everything

Limit amounts live in `LIMIT` (`stg_pds_limit`) and are only reachable through junctions — there is **no** `coverage_id` or `lob_id` on `LIMIT`, and `sourcetypeid` / `applicationtypeid` are 100% NULL in our data.

| Junction | Grain | Staging model |
|---|---|---|
| `PREMIUMLIMIT` (coverage → premium → limit) | rated coverage | `stg_pds_premium_limit` |
| `LOBRISKLIMIT` (risk_lob → limit) | risk | `stg_pds_lob_risk_limit` |
| `LOBLIMIT` (lob → limit) | LOB | `stg_pds_lob_limit` |
| `SUBCOVERAGELIMIT` (subcoverage → limit) | coverage line item | *not staged — empty at source* |

**Umbrella/excess underlying coverages are unrated**, so they have no `PREMIUMLIMIT` or `LOBRISKLIMIT` rows, and `LOBLIMIT` only carries the umbrella's own `Umbrella Limit`. The working link for underlying-coverage limits is the physical record key shared by `COVERAGE` and `LIMIT`:

```
COVERAGE.storage_id + COVERAGE.transaction_id + COVERAGE.record_number
  = LIMIT.storage_id + LIMIT.transaction_id + LIMIT.record_number
```

This key is unique per coverage and attaches each limit to its exact coverage, so limit types align to the coverage line automatically (GL → Each Occurrence / General Aggregate / Products-Completed Ops / Personal & Advertising Injury; Auto → Each Accident; Employers → Bodily Injury by Accident/Disease, Policy Limit By Disease; Other → Limit 1/2, Aggregate).

> **Validated 2026-07-30.** `stg_pds_coverage` now exposes `storage_id` and `record_number` alongside `transaction_id`, so this join is usable from staging. Confirmed against landed data:
> - The triple is **unique across all 757,189 coverages** — not just underlying ones.
> - It is the **only** route to underlying-coverage limits: of 10,271 Umbrella coverages, `PREMIUMLIMIT` reaches **0** and the physical key reaches **1,620**.
> - Column types match on both sides (`storage_id` TEXT, `transaction_id` NUMBER, `record_number` NUMBER), so no casting is needed.
>
> Only ~16% of umbrella coverages carry a limit even via this key, so expect most underlying coverages to have none rather than treating that as a defect. Nothing consumes the columns yet. (`CLAUDE.md` still cites an `rpt_excess_underlying_schedule` model that does not exist on this branch.)

## What is and isn't staged

Every PDS table we read is declared in `models/sources.yml` under source `pds` and has a matching `stg_pds_*` model in `models/1_stg/pds/`. Staging is one-to-one with source: rename to snake_case, cast, filter `where not _fivetran_deleted`, and add `dbt_utils.generate_surrogate_key()` keys.

Tables in the vendor diagram that are **not** staged or declared as sources — if a question needs them, they must be added to `sources.yml` first:

`BUREAU`, `STATEBUREAU`, `BUREAUFACTOR`, `TAXSURCHARGEFACTOR`, `RISKPARTY`, `ACCOUNTPARTY`, `SUBCOVERAGE`, `SUBCOVERAGELIMIT`.

`LOBFACTOR` **is** staged, as `stg_pds_lob_factor`, but is deliberately **diagnostic only and must stay
unconsumed** — reading it alongside the premium-grain judgement factor would multiply the underwriter's
load in twice and look entirely plausible. `assert_judgement_factor_reads_premium_grain_only` enforces
that.

Extended-data tables we do stage, which are not on the vendor diagram, carry customer-specific attributes as key/value or wide extensions: `COVERAGEEXTENDEDDATA`, `INSURANCEPARTYEXTENDEDDATA`, `LOBEXTENDEDDATA`, `LOBRISKEXTENDEDDATA`, `TRANSACTIONEXTENDEDDATA`.

## Query rules of thumb

1. **Pick one traversal pattern per query.** Mixing risk-level and LOB-level premium in one SELECT double-counts.
2. **Join transaction-scoped children on `policytransactionid`; keep detail rows scoped with `transactionid`.**
3. **Reach `RISK` through `STATE`**, and coverage/risk limits through `LOBRISK.risklobid`.
4. **One amount junction per query.** See the premium table above.
5. **Resolve a `*typeid` through `TYPE` when you want a label; hardcode it only when the id is a selector — and always comment it.** These are two different jobs:
   - **Label** — staging models join `TYPE` once to turn `*typeid` into `type_enum` / `type_description`, so downstream models read the label and never the id. Don't re-join `TYPE` downstream for something staging already exposes.
   - **Selector** — where a specific type *is* the filter or the pivot target, hardcoding the id is correct and unavoidable: `dim_lob` cannot produce a column named `limit_umbrella` without naming the type that feeds it, and the exposure models cannot filter to `limittypeid = 549` without naming it. About 23 models do this across ~95 sites.

   The failure mode to avoid is not hardcoding, it is an **unlabelled** magic number. Every hardcoded id must carry an inline comment with its `TYPE` label — `and lm.limit_type_id = 549 -- Exposure` — and where a report depends on it, a test asserting the id still resolves to that label. Documented sentinels like `statustypeid = 1474` (Previously Deleted) follow the same rule.
6. `POLICYHEADER` is one row per policy across all versions; point-in-time questions need `TRANSACTION` filtered by `transactioneffectivedate` / `commitdate`, or `currenttransactionid` for in-force.