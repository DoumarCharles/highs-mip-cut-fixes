# HiGHS MIP: invalid cuts from dropped tiny coefficients

Experimental patches for HiGHS (base `latest` 8fd6a9a) addressing false
`Optimal` claims that come from cut generation silently dropping very small
coefficients on columns with large bounds. Related upstream issue:
ERGO-Code/HiGHS#3434.

## The defect class

Several places in the MIP cut machinery remove coefficients with
`|a| <= small_matrix_value` without relaxing the right-hand side. When the
column's bound is large (1e6 and up), the dropped term `a * x` can exceed
`feastol`, so the resulting cut can remove feasible, even optimal, solutions.

## Patches

| file | what it does |
|---|---|
| `patches/fix1-untransform.patch` | `HighsTransformedLp::untransform`: relax the rhs for dropped tiny coefficients instead of dropping them silently (the #3434 fix). |
| `patches/fix4-v3.patch` | Fix 1, plus: the LP aggregator keeps all column terms (the aggregated row stays an exact equation; cut generation removes small coefficients with the correct directional relaxation); the tableau separator's dynamism filter ignores coefficients `<= small_matrix_value`; the path separator never substitutes out a column whose coefficient is only round-off (prevents an endless substitution loop); and the variable-bound branch of `HighsTransformedLp::transform` relaxes the rhs when it drops tiny coefficients. |
| `patches/fix4-v2.patch` | Alternative to v3: the aggregator drops tiny terms only on columns with finite bounds and returns the corresponding rhs relaxation to all separators. |

v3 is the smaller and preferred variant.

## Results so far

On a set of synthetic all-integer models with large bounds (several random
seeds each, `mip_rel_gap = 0`, a fixed time limit):

- Fix 1 alone leaves one false `Optimal` claim; an earlier fix-4 variant
  removed it but was much slower (tableau rows rejected by the dynamism filter
  because of kept round-off).
- v2 and v3 keep correctness on the checked cases and recover the speed:
  on the hardest 540 runs, 123 / 128 time limits for v2 / v3 against 108 for
  fix 1 alone and 185 for the earlier variant, with total time back to fix 1's
  level.
- `dcmulti` (presolve off) solves in the same time as fix 1 with v3.

Still to do before proposing anything upstream: exact false-claim check over
the full set for v2/v3, DEBUGSOL runs, unit tests and ctest, clean timing.

## Model

`models/mre.lp` with `models/mre-witness.sol` is the anonymised reproduction
model already attached to #3434.
