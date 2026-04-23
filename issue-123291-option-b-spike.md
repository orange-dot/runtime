# Issue 123291: Option B Feasibility Spike

Date: 2026-04-23
Repo: `/home/dev/work-base-20260421/workspace/platform/runtime`
Related investigation: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-jitdump-investigation.md`
Related decision memo: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-fix-decision.md`
Sibling spike: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-a-spike.md`

## Purpose

This spike evaluates **Option B** only:

- exploit the normalized tree shape created after the first diamond is if-converted
- avoid CFG-heavy matching on the original store/store diamond
- determine whether the fix can remain narrow instead of turning into a generic `SELECT` simplifier

This is not a patch. It is a source-and-evidence package for side-by-side comparison with Option A.

## Evidence Baseline

Authoritative dump:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt`

Primary source anchors:

- `IfConvertCheck()` flow and statement guards in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/ifconversion.cpp:61`
- select creation in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/ifconversion.cpp:497`
- `gtNewConditionalNode(GT_SELECT, ...)` fallback in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/ifconversion.cpp:512`
- `TryTransformSelectToOrdinaryOps()` in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/ifconversion.cpp:737`

Primary dump anchors:

- problematic method start: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:2698`
- `Optimize bools` no-op: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5104`
- `If conversion` start: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5171`
- post-`If conversion` `SELECT` store: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5225`
- final codegen: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:6680`

## Current Shape After `If conversion`

`If conversion` already rewrites the first diamond in the problematic method. The refreshed dump shows this concretely.

Before `If conversion`, the first half of the method is:

```text
BB01:
  JTRUE(EQ(arg0, 0)) -> BB03 / BB02

BB02:
  STORE_LCL_VAR tmp1 = GT(arg0, 0)

BB03:
  STORE_LCL_VAR tmp1 = 1
```

After `If conversion`, the first diamond is replaced with one store:

```text
BB01:
  STORE_LCL_VAR tmp1 = SELECT(EQ(arg0, 0), 1, GT(arg0, 0))
```

The dump shows the exact rewritten tree:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5225`

High-signal excerpt:

```text
STORE_LCL_VAR tmp1 =
  SELECT(
    EQ(arg0, 0),
    1,
    GT(arg0, 0))
```

The later control flow is still:

- `BB04`: `JTRUE(EQ(tmp1, 0))`
- `BB05`: side-effect call
- `BB06`: `RETURN tmp1`

So Option B does not need to rediscover the original store/store diamond. The normalized tree already exists.

## Why This Is A Real Option

`ifconversion.cpp` already contains an explicit “select to ordinary ops” escape hatch:

- `TryTransformSelectToOrdinaryOps()`

Today it already handles constant-boolean identities such as:

- `compare ? 1 : 0 -> compare`
- `compare ? 0 : 1 -> reverse(compare)`

That makes it a credible home for a narrow extension of the same style:

- `EQ(x, 0) ? 1 : GT(x, 0) -> GE(x, 0)`
- `EQ(x, 0) ? 1 : LT(x, 0) -> LE(x, 0)`

The important question for this spike is not whether a late hook exists. It does. The real question is whether the hook can stay narrow enough.

## Candidate Hook Point

The best Option B hook is not a very late consumer in lowering or codegen. It is the existing tree canonicalization point inside `If conversion` itself:

- extend `TryTransformSelectToOrdinaryOps()` or add a sibling helper that runs at the same point

Why this is the strongest B hook:

- it already receives `m_cond`, `trueInput`, and `falseInput`
- it is already authorized to replace `SELECT` with an ordinary non-`SELECT` tree
- it acts before the `GT_SELECT` fallback is committed
- it avoids backend-only fixes that would prove less about IR quality

Less attractive alternatives considered and rejected for the spike:

- a later `GT_SELECT` cleanup in `morph.cpp`
- a lowering-time special case in `lower.cpp`

Those alternatives would broaden the fix surface and weaken the “explicit `GE` / `LE` in IR” proof path.

## Narrow Matcher Contract

The narrowest B matcher can be expressed in terms of the three select ingredients already present at `TryTransformSelectToOrdinaryOps()`:

- `m_cond`
- `trueInput`
- `falseInput`

Required semantic shape:

- one arm is literal `1`
- the other arm is a signed compare against zero
- `m_cond` is the sibling zero compare on the same input
- the pair maps to an existing zero-family fold:
  - `EQ + GT -> GE`
  - `GT + EQ -> GE`
  - `EQ + LT -> LE`
  - `LT + EQ -> LE`

That is the semantic core. To keep this from becoming a general select simplifier, the v1 implementation should also carry local-use guards derived from the actual target family:

- the `SELECT` result is being materialized into a local via `GT_STORE_LCL_VAR`
- the local is the same bool temp later consumed by the target method tail:
  - `if (tmp) { side effect }`
  - `return tmp`

The important observation is that `optIfConvert()` already has:

- the destination local in `m_thenOperation.node`
- the merge block in `m_finalBlock`

That means Option B does not have to accept all matching compare-selects globally. It can, if desired, remain coupled to this specific materialized-bool family by checking the immediate post-conversion use structure.

## Expected Rewrite Shape

The direct rewrite target is:

```text
Before:
  STORE_LCL_VAR tmp =
    SELECT(EQ(x, 0), 1, GT(x, 0))

After:
  STORE_LCL_VAR tmp =
    GE(x, 0)
```

and similarly:

```text
SELECT(EQ(x, 0), 1, LT(x, 0)) -> LE(x, 0)
```

If operand order is reversed in the source expression, the normalized form is expected to swap which compare arrives as `m_cond` versus which compare arrives as the non-constant arm. The matcher therefore needs to treat the compare pair as unordered:

- `m_cond = EQ(x, 0)`, other arm `GT(x, 0)` -> `GE(x, 0)`
- `m_cond = GT(x, 0)`, other arm `EQ(x, 0)` -> `GE(x, 0)`
- same for the `LT` / `LE` family

## Variant Coverage

The target v1 family remains:

- `x == 0 || x > 0`
- `x > 0 || x == 0`
- `x == 0 || x < 0`
- `x < 0 || x == 0`

Expected normalized coverage:

- one compare becomes the `If conversion` condition
- the other compare becomes the non-constant `SELECT` arm
- literal `1` is the short-circuit-success arm

The important conclusion is that B can cover all four source variants with the same semantic matcher, but only if it deliberately treats the compare pair as unordered rather than tying the rewrite to `m_cond == EQ`.

## Scope-Control Risks

Option B is technically feasible, but its main risk is scope creep.

Without explicit guardrails, extending `TryTransformSelectToOrdinaryOps()` could silently become:

- a generic canonicalizer for compare-based selects
- a catch-all place for boolean materialization cleanup that really belongs elsewhere

The spike result is that this risk is manageable only if the implementation keeps all of the following boundaries:

- one arm must be literal `1`
- the other arm must be a signed zero compare
- the compare pair must map exactly to the existing `GE` / `LE` families
- the output must remain limited to the materialized-bool local family, not arbitrary value selects

This is implementable, but the scope discipline must be explicit in code and tests.

## Comparison With Option A

Option B has one clear advantage:

- it acts on a simpler tree than the original CFG diamond

It also has two structural disadvantages relative to Option A:

- the semantic rule already lives in `optimizebools.cpp`, so B would duplicate part of that ownership in `ifconversion.cpp`
- if its local-use guard is weakened later, B naturally wants to expand into broader select simplification

Neither point disqualifies B. They do matter in the final comparison.

## Proof Path

If Option B is selected, the proof path is still acceptable:

1. add regression coverage in `src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`
2. rerun the checked dump harness
3. show that the `SELECT(...)` materialization no longer exists for the target family
4. show explicit `GE` / `LE` in IR before lowering/codegen

This proof path is slightly less direct conceptually than Option A because the rule lives in a different phase than the original compare-fold logic, but it still produces explicit IR evidence.

## Feasibility Conclusion

Option B is feasible.

The most credible implementation hook is:

- `TryTransformSelectToOrdinaryOps()` in `ifconversion.cpp`

The route to explicit `GE` / `LE` is concrete, because the normalized inputs are already present there. The main engineering question is not feasibility; it is whether the rule feels phase-local enough compared with Option A once both spikes are reviewed together.

This spike therefore keeps Option B alive as a real candidate, with a credible narrow hook and no unresolved source-level blocker.
