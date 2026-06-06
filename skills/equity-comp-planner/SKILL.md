---
name: equity-comp-planner
version: 1.0.0
description: Value and tax employer equity and assess single-stock concentration by orchestrating the public planfi MCP. Use whenever someone wants to understand RSUs / ISOs / NSOs / ESPP, find how many ISOs they can exercise before AMT kicks in, or decide whether they hold too much employer stock and what diversifying would cost in tax — e.g. "value my 5,000 RSUs vesting this year", "when does AMT kick in if I exercise 5,000 ISOs at a $12 strike, FMV $40?", "$600k of my $1M net worth is in Acme stock — should I diversify?".
---

# Equity Comp Planner

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All equity valuation, AMT/tax math, and concentration thresholds live server-side. This skill only
gathers inputs and calls the tools — it does **not** compute anything locally and bakes in no
defaults of its own. Read-only.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_equity_compensation`):
`analyze_equity_compensation`, `analyze_iso_amt_crossover`, `analyze_stock_concentration`, plus
optional `generate_financial_plan` (for `plan_id` chaining + a `share_url`). Use whichever name your
environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. All three equity tools accept `{ plan_id }` (plus inline
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
`RSU` | `ISO` | `NSO` | `ESPP`. Optional per-grant fields the server accepts vary by type (e.g.
current FMV / `current_price_per_share`, vesting schedule, ESPP discount). Optional top-level:
`current_price_per_share`, `ordinary_tax_rate`, `capital_gains_tax_rate`, `filing_status`, `age`.

```
analyze_equity_compensation({
  grants: [
    { type: "RSU",  shares: 5000, grant_price_per_share: 0 },
    { type: "ISO",  shares: 2000, grant_price_per_share: 12 }
  ],
  current_price_per_share: 40
})
```

### "How many ISOs can I exercise before AMT?" → `analyze_iso_amt_crossover`
Self-orchestrating: returns the `crossover_bargain_element` (and shares) you can exercise-and-hold
before triggering AMT, the `amt_at_crossover`, and a sensitivity table around it.
Useful fields: `iso_shares`, `strike_price` (a.k.a. `exercise_price_per_share`),
`current_fmv` (a.k.a. `fmv_per_share`), `baseline_ordinary_income`, `filing_status`,
`state_flat_rate`. Runs from sparse input — missing fields fall back to server defaults reported as
prose (see Step 3).

```
analyze_iso_amt_crossover({
  iso_shares: 5000, strike_price: 12, current_fmv: 40,
  baseline_ordinary_income: 200000, filing_status: "single"
})
```

### "Do I have too much in employer stock?" → `analyze_stock_concentration`
Concentration ratio vs liquid net worth, risk flag, a suggested diversification glidepath, and the
tax cost of unwinding the position.
REQUIRED: `position_value`, `liquid_net_worth`. Optional: `tax_lots[]` — each
`{ basis, acquisition_date }` (lets the server split long- vs short-term gains and stage sales),
`capital_gains_tax_rate`, `cost_basis`, `annual_diversification_budget`.

```
analyze_stock_concentration({
  position_value: 600000, liquid_net_worth: 1000000,
  tax_lots: [{ basis: 120000, acquisition_date: "2022-03-01" }]
})
```

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — total equity value + by-grant tax treatment; the ISO AMT-crossover
  shares / bargain element; or the concentration ratio + risk flag + tax cost to diversify.
- **Read back `disclosures.key_assumptions`** verbatim. These tools do **not** emit a structured
  `assumed_defaults[]` array — instead they apply silent Zod defaults (e.g. ordinary / cap-gains
  rates, filing status, FMV when not supplied) and expose what they assumed only as **prose in
  `disclosures.key_assumptions`**. Surface those so the user can correct any silent assumption.
- Honor `disclosures.not_advice` — present as planning estimates, not tax advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }`
  when available). These commonly chain **into the tax-optimizer skill** — e.g. `optimize_multi_year_tax`
  (time the ISO exercise + spread the AMT hit across years) or `analyze_advanced_taxes` (full
  NIIT/AMT/state bite on a same-year exercise or large diversifying sale). Use the server-suggested
  chains rather than guessing the next call.
- **For a share link:** the equity tools don't return one. If the user wants a sharable plan, run
  `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the three equity tools (with `{ plan_id }` or raw fields).
3. Read back the headline + `disclosures.key_assumptions`.
4. Follow `next_actions[]` — frequently chains into the **tax-optimizer** tools
   (`optimize_multi_year_tax` / `analyze_advanced_taxes`) to time the exercise and quantify the
   surtax bite, or `generate_financial_plan` for a share link.

## Fictional examples

**1.** *"I have 5,000 ISOs at a $12 strike, FMV is $40 — when does AMT kick in?"*
→ `analyze_iso_amt_crossover({ iso_shares: 5000, strike_price: 12, current_fmv: 40 })`. Lead with
the crossover shares / bargain element and the AMT owed if exercised beyond it; offer to chain
`optimize_multi_year_tax` to spread the exercise across tax years. Read back key_assumptions
(baseline income, filing status, state rate).

**2.** *"$600k of my $1M net worth is in Acme stock — should I diversify, and what's the tax hit?"*
→ `analyze_stock_concentration({ position_value: 600000, liquid_net_worth: 1000000 })`. Lead with
the 60% concentration ratio + risk flag and the tax cost of unwinding; offer `tax_lots[]` for a
staged long-vs-short-term sale plan, and chain `analyze_advanced_taxes` for the NIIT bite on a large
realizing sale. Read back key_assumptions (cap-gains rate, cost basis).

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits are ~2026.
- `analyze_equity_compensation` and `analyze_stock_concentration` have REQUIRED inputs (`grants[]`;
  `position_value` + `liquid_net_worth`) — ask for them before calling. `analyze_iso_amt_crossover`
  self-orchestrates and runs from sparse input.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- These three tools surface assumptions as **prose in `disclosures.key_assumptions`**, not a
  structured `assumed_defaults[]`, and they do **not** return a `share_url` — chain
  `generate_financial_plan` for a sharable link. (Server follow-up tracked in `SKILL_AUTHORING.md`.)
- Equity moves are tax-driven — the natural next step is usually the **tax-optimizer** skill.
- Not financial or tax advice. Planning estimates only.
