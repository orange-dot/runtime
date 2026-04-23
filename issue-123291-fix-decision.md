# Issue 123291: Fix Decision Memo

Date: 2026-04-23
Repo: `/home/dev/work-base-20260421/workspace/platform/runtime`
Related investigation: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-jitdump-investigation.md`
Option A spike: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-a-spike.md`
Option B spike: `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-b-spike.md`

## Purpose

This memo started as the decision gate for fixing the `dotnet/runtime#123291` missed optimization.

A later local implementation/proof pass selected **Option A** for PR preparation while keeping the earlier A-vs-B spike comparison as design context.

This memo now serves three roles:

- lock the evidence baseline used by both spikes
- record the local proof closure for the chosen option
- keep the earlier comparison and scope controls available for maintainer review

## Current Technical State

The existing JIT already knows how to fold:

- `x == 0 || x > 0` to `x >= 0`
- `x == 0 || x < 0` to `x <= 0`

That semantic rule already exists in:

- `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp`

The target failure is not missing semantics. It is missed shape recognition for the materialized-bool family:

- `bool b = x == 0 || x > 0;`
- `if (b) { side effect }`
- `return b;`

From the refreshed checked dump baseline:

- direct-return forms already reach explicit `GE`
- the materialized-bool family reaches `Optimize bools` without triggering
- `If conversion` later rewrites the first diamond into `SELECT(...)`
- no later phase canonicalizes that `SELECT(...)` to explicit `GE` / `LE`

Authoritative evidence baseline:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt`

Authoritative post-patch proof artifact:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt`

The older `/tmp/dotnet-jit-repro/full-jitdump-mix.txt` is not the required baseline for this decision work.

## Updated Local Status

Option A is now implemented locally in:

- `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp`
- `/home/dev/work-base-20260421/workspace/platform/runtime/src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`

The local proof package is closed:

- checked post-patch `JitDump` shows `fgOptimizeMaterializedBoolCompare` firing and replacing the first diamond with `STORE_LCL_VAR tmp1 = GE(arg0, 0)`:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12509`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12530`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12531`
- final checked codegen for `Program:IsGreaterThanOrEqualZero(int):bool` is now 20 bytes:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13776`
- official checked subset build for `JIT/opt/OptimizeBools` succeeds:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/log/TestBuild.linux.x64.Checked.log:104`
  - `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/log/TestBuild.linux.x64.Checked.log:105`
- checked xUnit execution succeeds for both built tests:
  - `/tmp/optboolsreturn.testResults.xml:3`
  - `/tmp/Runtime_123621.testResults.xml:3`

The only remaining open question for PR discussion is maintainer preference on phase ownership. There is no longer a local evidence gap about whether Option A works or whether the regression subset can be built and run.

## Scope

The intended first fix remains restricted to the medium family only:

- `x == 0 || x > 0`
- `x > 0 || x == 0`
- `x == 0 || x < 0`
- `x < 0 || x == 0`

And only for the materialized-bool-plus-`if`-plus-`return` family.

Still out of scope unless they fall out for free:

- `!= 0 && >= 0` or `<= 0` families
- generic bool-local cleanup
- generic `SELECT` simplification
- VM, importer, or non-JIT work

## Decision Inputs

The option analysis now lives in two separate spike packages:

- Option A spike:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-a-spike.md`
- Option B spike:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-b-spike.md`

Both spikes are expected to be implementation-grade:

- exact source hooks
- exact before/after IR or CFG shape
- explicit narrowing invariants
- operand-order coverage
- proof route to explicit `GE` / `LE`

## Review Gate

The recommendation is blocked on an explicit review step:

- Spike A must be reviewed first by the requester before any option is chosen

That review gate exists to validate:

- evidence quality
- narrowness of the proposed matcher
- whether the A/B comparison is being made on equal technical footing

Producing Spike B does not by itself authorize a recommendation. The option remains unselected until the requester reviews Spike A and the two spikes are compared directly.

This review gate is now informational rather than blocking for local implementation. The local patch already proceeds with Option A; the maintainer review should focus on whether they agree with keeping the fold owned by `optimizebools.cpp`.

## Decision Rule

After Spike A review, compare Option A and Option B using the same rubric:

- smaller matcher surface
- clearer phase-local ownership
- fewer new invariants beyond what the target phase already assumes
- more natural coverage of operand-order variants
- more direct proof path to explicit `GE` / `LE`
- lower risk of broadening into unrelated optimization work

Tie-break rule:

- prefer the option that stays closest to the existing compare-fold semantics already owned in `optimizebools.cpp`

A decision must not be justified only by “cleaner final asm”. The chosen path must have a defensible route to explicit `GE` / `LE` in the IR used for validation.

## Acceptance Criteria For The Eventual Patch

The final implementation is accepted only if all of the following are true:

- functional regression coverage is added for the target materialized-bool family
- existing direct-return cases continue to pass
- the target shape reaches explicit `GE` or `LE` in local checked JIT evidence
- the implementation stays within the chosen option’s declared scope

Required proof artifacts:

- regression coverage in `src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`
- local checked `JitDump` or `JitDisasm` showing explicit `GE` or `LE`

## Post-Decision Backlog Shape

Once one option is selected, the implementation backlog is already defined:

1. implement the chosen matcher only for the medium family
2. add or update regression coverage in `src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`
3. rerun the checked dump harness
4. capture explicit `GE` / `LE` proof for the target family
5. confirm existing direct-return coverage is unchanged

The non-selected option remains backlog context only. It does not become parallel implementation work.

## Local Recommendation

Select **Option A** for the PR.

Current local state:

- Spike A is documented
- Spike B is documented
- Option A is implemented locally
- checked `JitDump` proof is closed
- checked `OptimizeBools` subset build/run proof is closed

Maintainer review can still redirect the final upstream shape, but the local default recommendation is no longer open-ended:

- keep the fold in `optimizebools.cpp`
- keep the scope limited to the medium family and the materialized-bool-plus-`if`-plus-`return` shape
- present Option B only as fallback context, not as parallel implementation work
