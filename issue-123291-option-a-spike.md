# Issue 123291: Option A Feasibility Spike

Date: 2026-04-23
Repo: `/home/dev/work-base-20260421/workspace/platform/runtime`
Related investigation: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-jitdump-investigation.md`
Related decision memo: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-fix-decision.md`
Sibling spike: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-b-spike.md`

## Purpose

This spike evaluates **Option A** only:

- fix the missed optimization before `If conversion`
- keep the work owned by `optimizebools.cpp`
- prove whether the materialized-bool-plus-`if`-plus-`return` family can be matched narrowly enough for implementation

This is not a patch. It is a source-and-evidence package meant to let an implementer start immediately once the option is selected.

## Evidence Baseline

The current authoritative dump is:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt`

The older `/tmp/dotnet-jit-repro/full-jitdump-mix.txt` was an earlier temporary output and is not required for this spike. All conclusions below are grounded in the refreshed `full-jitdump-all-123291.txt` plus current source.

Primary source anchors:

- existing `EQ + GT -> GE` / `EQ + LT -> LE` fold in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:222`
- `optOptimizeBools()` driver in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:1593`
- `fgFoldCondToReturnBlock()` in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:1691`
- target regression file `/home/dev/work-base-20260421/workspace/platform/runtime/src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`

Primary dump anchors:

- problematic method start: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:2698`
- `Optimize bools` no-op: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5104`
- `If conversion` start: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5171`
- final codegen for the problematic method: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:6680`
- direct-return `GE` proof baseline: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:8684`

## Current Shape At The `Optimize bools` Boundary

By the time `optOptimizeBools()` runs, the problematic method is no longer the raw import-time six-block shape with a second temp for `dup`. Copy propagation and SSA cleanup have already reduced the live boolean state to a single temp local `V02 tmp1`, but the method is still fundamentally split across four roles:

- entry compare block
- two predecessor stores into the same bool temp
- later `if (tmp)` block
- final `return tmp` block

The high-signal shape immediately before `If conversion` is:

```text
BB01:
  JTRUE(EQ(arg0, 0)) -> BB03 / BB02

BB02:
  STORE_LCL_VAR tmp1 = GT(arg0, 0)
  goto BB04

BB03:
  STORE_LCL_VAR tmp1 = 1
  goto BB04

BB04:
  JTRUE(EQ(tmp1, 0)) -> BB06 / BB05

BB05:
  CALL side_effect
  goto BB06

BB06:
  RETURN tmp1
```

Concrete dump evidence at `Optimize bools`:

- `BB02` store of `GT(arg0, 0)`: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5120`
- `BB03` store of constant `1`: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5129`
- `BB04` branch on `EQ(tmp1, 0)`: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5136`
- `BB06` return of `tmp1`: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5147`

`optOptimizeBools()` reports:

- `optimized 0 BBJ_COND cases in 1 passes`

That is the exact failure surface Option A needs to close.

## Why Existing Option-A-Adjacent Code Misses

There are two nearby optimization entry points today, and the target shape satisfies neither.

### `fgFoldCondToReturnBlock()`

Current preconditions in `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:1691` require:

- the current block is `BBJ_COND`
- both successors are `BBJ_RETURN`
- both return blocks are literal `return true` / `return false`

The problematic method does not look like that:

- `BB01` branches to two store blocks, not return blocks
- `BB04` branches to `return tmp` versus `call; return tmp`, not literal `return 1/0`

So the current return-fold path never even begins.

### `optOptimizeBoolsCondBlock()`

The `optOptimizeBools()` driver only calls the two-block boolean fold when `b2 = b1->GetFalseTarget()` is itself `BBJ_COND`.

That works for direct compare chains like:

- `cond -> cond`

The problematic method at the first diamond is instead:

- `cond -> always(store tmp)`

So it also falls outside the existing compare-fold path even though the semantic pair is the same `EQ + GT` family that already maps to `GE`.

## Candidate Hook Point

The narrowest hook for Option A is:

- add a new recognizer in `optimizebools.cpp`, called from `optOptimizeBools()`, adjacent to the existing `fgFoldCondToReturnBlock()` / `optOptimizeBoolsCondBlock()` checks

The important point is not the exact helper name, but the ownership boundary:

- the matcher should live in the same phase that already owns boolean compare folding
- it should fire before `If conversion`
- it should reuse the same semantic mapping already encoded for direct-return folding:
  - `EQ + GT -> GE`
  - `EQ + LT -> LE`

The most natural place in the current driver is after the `BBJ_COND` check and before the `b2->KindIs(BBJ_COND)` gate decides there is nothing to do with the first diamond.

## Narrow Matcher Contract

The matcher can stay narrow if it is expressed in terms of block roles and local-def/use roles, not exact block numbers.

Required preconditions:

- current method returns `TYP_UBYTE`
- `b1` is `BBJ_COND`
- both successors of `b1` are simple store blocks with:
  - unique predecessor `b1`
  - same unique successor `join`
  - one side storing literal `1`
  - the other side storing a compare result
- both stores target the same local `tmp`
- the compare side stores one of:
  - `GT(x, 0)`
  - `LT(x, 0)`
  - or the operand-order equivalent compare that still maps to the same signed zero family
- the compare in `b1` is the sibling zero compare on the same `x`
- the later flow from `join` is the target family only:
  - `join` is `BBJ_COND`
  - `join` tests `tmp == 0`
  - one successor is a side-effect block with unique predecessor `join`
  - the final return block returns the same `tmp`

Scope-control guards that keep this from opening a broader boolean-local optimization:

- same local on both predecessor stores and the final return
- one predecessor must store literal `1`
- compare pair must be exactly the existing zero-family fold, not arbitrary boolean expressions
- signed compare only; do not infer `GE` / `LE` from unsigned zero compares
- no extra statements in the two predecessor store blocks beyond the single store and any ignorable `NOP`

## Expected Rewrite Shape

The cleanest implementation route is not to invent a new late canonical form. It is to collapse the first diamond into a single store of the already-supported explicit compare:

```text
Before:
  tmp = (x == 0) ? 1 : (x > 0)

After:
  tmp = (x >= 0)
```

In CFG terms:

```text
Before:
  BB01 cond -> BB02/BB03
  BB02 store tmp = GT(x, 0)
  BB03 store tmp = 1
  BB04 if (tmp == 0) ...
  BB06 return tmp

After:
  BB01 always:
    STORE_LCL_VAR tmp = GE(x, 0)
  BB04 if (tmp == 0) ...
  BB06 return tmp
```

This is enough to satisfy the proof requirement:

- explicit `GE` / `LE` exists in IR before later lowering/codegen
- later `if (tmp)` and `return tmp` can remain unchanged in the first implementation

The implementation does not need to solve the side-effect block and the return block in the same transform. It only needs to canonicalize the materialized bool correctly and safely.

## Variant Coverage

The intended v1 family is:

- `x == 0 || x > 0`
- `x > 0 || x == 0`
- `x == 0 || x < 0`
- `x < 0 || x == 0`

Coverage expectation for the matcher:

- `EQ + GT` on the same `x` maps to `GE`
- `GT + EQ` on the same `x` also maps to `GE`
- `EQ + LT` on the same `x` maps to `LE`
- `LT + EQ` on the same `x` also maps to `LE`

This is the same semantic table already embodied by the existing compare-fold logic. The new work is shape recognition, not a new algebraic rule.

## Safety And Brittleness Risks

Primary risks for Option A:

- overfitting to one exact CFG layout instead of matching role-based invariants
- accidentally accepting non-bool locals or broader “materialized compare” patterns
- coupling the transform to temporary block numbering instead of def/use structure

The spike result is that none of these risks are blockers if the matcher stays role-based and local-use constrained as listed above.

What should stay out of scope in the first patch:

- `!= 0 && >= 0` or `<= 0` families
- arbitrary `bool tmp` materialization cleanup
- generic “cond-store-cond-return” folding
- VM, importer, or backend work

## Proof Path

If Option A is selected, the implementation proof path is straightforward:

1. add regression coverage in `src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`
2. rerun the checked dump harness
3. show that the problematic materialized-bool method now contains explicit `GE` or `LE` in JIT IR before final lowering/codegen

The proof does not rely on final asm aesthetics alone. The IR itself becomes the acceptance artifact.

## Feasibility Conclusion

Option A is feasible without widening ownership beyond `optimizebools.cpp`.

The narrowest viable hook is a new recognizer in `optOptimizeBools()` for:

- `cond`
- two same-local predecessor stores
- later `if (tmp)` plus `return tmp`

The route to explicit `GE` / `LE` is direct, because the compare fold already exists; the missing piece is only reaching it from this CFG/local-materialization shape.

This spike leaves the final option choice open. It does establish that Option A has a concrete, implementation-grade path with no unresolved source-level blocker.
