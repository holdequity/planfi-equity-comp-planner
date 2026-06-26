---
name: equity-comp-planner
version: 1.0.2
description: Value and tax employer equity and assess single-stock concentration by orchestrating the public planfi MCP. Use whenever someone wants to understand RSUs / ISOs / NSOs / ESPP, find how many ISOs they can exercise before AMT kicks in, or decide whether they hold too much employer stock and what diversifying would cost in tax — e.g. "value my 5,000 RSUs vesting this year", "when does AMT kick in if I exercise 5,000 ISOs at a $12 strike, FMV $40?", "$600k of my $1M net worth is in Acme stock — should I diversify?".
---

# Equity Comp Planner

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All equity valuation, AMT/tax math, and concentration thresholds live server-side. This skill only
gathers inputs and calls the tools — it does **not** compute anything locally and bakes in no
defaults of its own. Read-only.

**Related skills:** for the pre-IPO/founder-specific 83(b) early-exercise election and the §1202 QSBS
gain exclusion, see **startup-equity-planner** (`analyze_startup_equity`). For an employer
Nonqualified Deferred Comp (NQDC / 409A) election — deferring salary/bonus past qualified-plan caps,
lump-vs-installment distribution, and creditor risk — see **deferred-comp** (`analyze_deferred_comp`).
For **appreciated employer stock held inside a 401(k)** at job change or retirement — whether to
elect a Net Unrealized Appreciation (NUA) in-kind lump-sum distribution (LTCG on the appreciation)
versus an IRA rollover or Roth conversion — use `analyze_nua` (Step 2 below).

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_equity_compensation`):
`analyze_equity_compensation`, `analyze_iso_amt_crossover`, `analyze_stock_concentration`,
`analyze_nua`, plus optional `generate_financial_plan` (for `plan_id` chaining + a `share_url`). Use whichever name your
environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — (Optional) build a plan first to chain context + get a share link

> **Feed it into the forecast (not just plan_id chaining):** `generate_financial_plan` now accepts `equity_compensation` directly as a plan input, so it flows into net worth, FIRE %, and Monte-Carlo backtesting — vested after-tax shares vest into the portfolio. Use the standalone analyze tool below for a focused what-if; pass `equity_compensation` into the plan to see its effect on the whole household forecast.

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. All equity tools accept `{ plan_id }` (plus inline
overrides), so they can resolve age, income, filing status, and net worth from the saved plan
instead of you re-sending every figure. `generate_financial_plan` also returns a **`share_url`**
(planfi.app) — the equity tools themselves do **not** emit a share link, so this is the way to give
the user one.

This step is optional: every equity tool runs from raw inputs too. Prefer the plan path when the
session already has a model or the user wants a sharable artifact.

> **Engine facts to bake in:** all decimals are **fractions** (24% → `0.24`); all dollars are
> **today's (real) dollars**; brackets/limits are approximate **~2026** values.

## Step 2 — Route by intent

Pick the tool that matches what the user is asking. Pass `{ plan_id }` when you have one; otherwise
pass the raw fields below.

### "Value / tax my RSUs, ISOs, NSOs, or ESPP" → `analyze_equity_compensation`
Per-grant value plus the ordinary-income / capital-gains / AMT-preference tax treatment of each
grant type, and total taxable position.
REQUIRED: `grants[]` — each `{ type, shares, grant_price_per_share }` where `type` is one of
`RSU` | `ISO` | `NSO` | `ESPP`, `shares` is the total granted, and `grant_price_per_share` is the
current/grant fair-market value per share. Optional per-grant fields: `name`, `ticker`,
`annual_growth_rate` (real share-price growth), `grant_year`, `vesting_years`, `cliff_years`,
`strike_price_per_share` (options), `supplemental_withholding_rate`, `espp_discount`,
`espp_qualifying_disposition`, `espp_offering_price_per_share`, `espp_sale_price_per_share`,
`iso_exercise_and_hold`. Optional top-level: `ordinary_tax_rate`, `long_term_cap_gains_rate`,
`amt_rate`, `auto_sell_enabled`, `auto_sell_fraction`, `tax_year`, `plan_id`, `overrides`. (There is
no top-level `current_price_per_share` / `filing_status` / `age` field — the model is grant-driven.)

```
analyze_equity_compensation({
  grants: [
    { type: "RSU",  shares: 5000, grant_price_per_share: 40 },
    { type: "ISO",  shares: 2000, grant_price_per_share: 40, strike_price_per_share: 12 }
  ]
})
```

### "How many ISOs can I exercise before AMT?" → `analyze_iso_amt_crossover`
Returns the `crossover_bargain_element` you can exercise-and-hold before triggering AMT, the
`amt_at_crossover`, an optional `shares_estimate`, and a sensitivity table around it.
REQUIRED: `ordinary_taxable_income` (ordinary taxable income after deductions; drives the
regular-tax baseline and AMTI). Optional: `filing_status` (`single` | `married_joint`),
`per_share_bargain` (FMV − strike per share — supply it to turn the crossover bargain element into a
`shares_estimate`), `tax_year`, `plan_id`, `overrides`. Runs from sparse input — omitted optional
fields fall back to server defaults reported in `assumed_defaults[]` (see Step 3).

```
analyze_iso_amt_crossover({
  ordinary_taxable_income: 200000,
  per_share_bargain: 28,
  filing_status: "single"
})
```

### "Do I have too much in employer stock?" → `analyze_stock_concentration`
Concentration ratio vs liquid net worth, risk flag, a suggested diversification glidepath, and the
tax cost of unwinding the position.
REQUIRED: `position_value`, `liquid_net_worth`. Optional: `ticker`, `cost_basis`,
`warning_threshold_percent`, `target_percent`, `sell_down_years`, `long_term_cap_gains_rate`,
`niit_rate`, `state_tax_rate`, `stress_drop_percent`, `lot_method` (`HIFO` | `FIFO` | `SPECID`),
`short_term_rate`, `tax_year`, `plan_id`, `overrides`, and `tax_lots[]` — each
`{ basis, acquisition_date, market_value?, shares? }` where `basis` and `acquisition_date` are
required (lets the server split long- vs short-term gains and stage sales).

```
analyze_stock_concentration({
  position_value: 600000, liquid_net_worth: 1000000,
  tax_lots: [{ basis: 120000, acquisition_date: "2022-03-01" }]
})
```

### "Should I do NUA / net unrealized appreciation on my 401(k) company stock?" / "I'm retiring/leaving with appreciated employer stock in my 401(k) — lump-sum distribution vs IRA rollover?" / "Can I pay LTCG instead of ordinary rates on my company stock?" / "NUA vs Roth conversion on concentrated employer stock" → `analyze_nua`

**Always CALL `analyze_nua` for these — do not answer from general knowledge or quote NUA rules of thumb from memory. When the user gives the cost basis and market value, run it and lead with its real output (the NUA-vs-rollover breakeven and recommended election).**

Models the in-kind lump-sum distribution election: cost basis is taxed at ordinary rates now, while
the Net Unrealized Appreciation is taxed at LTCG (0/15/20 + NIIT) at sale — vs rolling the whole
position to an IRA and paying ordinary rates on every future withdrawal. Returns the tax owed on
each path, the `nua_advantage`, the `breakeven_ordinary_rate`, and a recommended election. Also
compares a Roth-conversion alternative when `roth_conversion_amount` is supplied.
REQUIRED: `cost_basis` (plan basis in the employer stock), `market_value` (FMV of the shares at
distribution). Optional: `ordinary_taxable_income` (other ordinary income in the distribution year —
NUA stacks on top), `filing_status` (`single` | `married_joint`), `expected_holding_years`,
`expected_growth_rate` (real post-distribution appreciation), `state_tax_rate`,
`roth_conversion_amount` (sizes the Roth-conversion branch), `tax_year`, `plan_id`, `overrides`.

```
analyze_nua({
  cost_basis: 100000,
  market_value: 1000000,
  ordinary_taxable_income: 250000,
  filing_status: "married_joint"
})
```

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — total equity value + by-grant tax treatment; the ISO AMT-crossover
  shares / bargain element; the concentration ratio + risk flag + tax cost to diversify; or the
  NUA-vs-IRA-rollover advantage + breakeven ordinary rate + recommended election.
- **Read back `assumed_defaults[]`** — all four tools build and return a structured
  `assumed_defaults[]` array of `{ field, assumed_value, note }` for every input they defaulted
  (e.g. ordinary / cap-gains / AMT rates, filing status, cost basis, vesting schedule, stress drop).
  Read these back so the user can correct any silent assumption. (`disclosures.key_assumptions` is a
  SEPARATE list of static methodology prose — surface it too if useful, but the per-field defaults
  the user can correct live in `assumed_defaults[]`.)
- Honor `disclosures.not_advice` (a **boolean** flag, not a message) — present results as planning
  estimates, not tax advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }`
  when available). Only `analyze_iso_amt_crossover` currently emits an edge: → `analyze_advanced_taxes`
  (feed the crossover bargain element into the full AMT/regular-tax model). `analyze_nua` emits edges
  → `analyze_roth_conversion` (compare the Roth-conversion alternative), `analyze_advanced_taxes`
  (quantify the full ordinary/LTCG/NIIT/state bite of the election), and
  `analyze_stock_concentration` (diversify the now-taxable NUA shares). `analyze_equity_compensation`
  and `analyze_stock_concentration` emit an empty `next_actions[]` — for those, suggest the
  tax-optimizer tools manually if relevant. Use the server-suggested chain when present rather than
  guessing.
- **For a share link:** the equity tools don't return a `share_url`. If the user wants a sharable
  plan, run `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the four equity tools (with `{ plan_id }` or raw fields).
3. Read back the headline + `assumed_defaults[]` (and `disclosures.key_assumptions` for methodology).
4. Follow `next_actions[]` when present — `analyze_iso_amt_crossover` chains to `analyze_advanced_taxes`
   (quantify the full AMT/NIIT/state bite). For the other two tools `next_actions[]` is empty; suggest
   the **tax-optimizer** tools manually if relevant, or `generate_financial_plan` for a share link.

## Fictional examples

**1.** *"I have 5,000 ISOs at a $12 strike, FMV is $40 — when does AMT kick in?"*
→ `analyze_iso_amt_crossover({ ordinary_taxable_income: 200000, per_share_bargain: 28, filing_status: "single" })`
(the $28 per-share bargain = $40 FMV − $12 strike; pass it to get a `shares_estimate`). Lead with
the crossover bargain element / share estimate and the AMT owed if exercised beyond it; follow the
`analyze_advanced_taxes` next-action to quantify the full bite. Read back `assumed_defaults[]`
(filing status, tax year).

**2.** *"$600k of my $1M net worth is in Acme stock — should I diversify, and what's the tax hit?"*
→ `analyze_stock_concentration({ position_value: 600000, liquid_net_worth: 1000000 })`. Lead with
the 60% concentration ratio + risk flag and the tax cost of unwinding; offer `tax_lots[]` for a
staged long-vs-short-term sale plan. Read back `assumed_defaults[]` (cost basis assumed $0,
cap-gains / NIIT / state rates, sell-down horizon).

**3.** *"I'm retiring with $750k of company stock in my 401(k) that I bought for $90k — should I elect NUA or roll it to an IRA?"*
→ `analyze_nua({ cost_basis: 90000, market_value: 750000, ordinary_taxable_income: 220000, filing_status: "married_joint" })`.
Lead with the real output — the NUA path pays ordinary tax on the $90k basis now plus LTCG (0/15/20
+ NIIT) on the $660k appreciation at sale, vs ordinary rates on the full IRA-rollover withdrawal;
surface `nua_advantage`, the `breakeven_ordinary_rate`, and the recommended election. Follow the
`analyze_roth_conversion` / `analyze_advanced_taxes` next-actions; read back `assumed_defaults[]`
(future ordinary rate, growth rate, state rate). Add `roth_conversion_amount` to size the
Roth-conversion alternative.

*(All examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits are ~2026.
- `analyze_equity_compensation` and `analyze_stock_concentration` have REQUIRED inputs (`grants[]`;
  `position_value` + `liquid_net_worth`) — ask for them before calling. `analyze_iso_amt_crossover`
  self-orchestrates and runs from sparse input.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- All four tools return a structured `assumed_defaults[]` array (`{ field, assumed_value, note }`)
  for every defaulted input — read it back so the user can correct silent assumptions.
  `disclosures.key_assumptions` is separate methodology prose, and `disclosures.not_advice` is a
  boolean flag. The equity tools do **not** return a `share_url` — chain `generate_financial_plan`
  for a sharable link.
- Equity moves are tax-driven — the natural next step is usually the **tax-optimizer** skill.
- Not financial or tax advice. Planning estimates only.
