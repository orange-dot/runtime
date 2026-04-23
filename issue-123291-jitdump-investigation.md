# Issue 123291: JitDump investigation notes

Date: 2026-04-23
Repo: `/home/dev/work-base-20260421/workspace/platform/runtime`
Branch/HEAD at start of investigation: `main`, commit `9ffefca3`

## Goal

Get a full checked `JitDump` for the repro around `dotnet/runtime#123291`, without making source changes, and confirm whether the issue is:

- a missing optimization rule, or
- an existing rule that does not trigger for this IR shape.

## High-signal result

A full checked `JitDump` was obtained for the target methods.

Follow-up fix memo:

- `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-fix-decision.md`
- `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-a-spike.md`
- `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-b-spike.md`

Main dump files:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt`

Current authoritative baseline:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt`

The earlier `/tmp/dotnet-jit-repro/full-jitdump-mix.txt` was a temporary intermediate dump from an earlier run and is not required for the current decision work.

Key conclusion from the dump:

- The rule `x == 0 || x > 0 -> x >= 0` already exists.
- The problematic `bool local + if + return` shape does not get folded into `GE` for the tested method.
- `optOptimizeBools()` reports `optimized 0 BBJ_COND cases in 1 passes` for the problematic method.
- The issue looks like a missed shape / missed trigger path, not an absent optimization rule.

## Relevant source references

- Existing fold rule:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:222`
- `optOptimizeBools` phase:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:1593`
- `fgFoldCondToReturnBlock` path:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:1691`
  - `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp:1718`
- `If conversion` path that later rewrites store/store diamonds:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/ifconversion.cpp:372`
- `JitDump` docs:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/docs/design/coreclr/jit/viewing-jit-dumps.md:1`

## Repro program

Temporary repro project:

- `/tmp/dotnet-jit-repro/Program.cs`
- `/tmp/dotnet-jit-repro/dotnet-jit-repro.csproj`

Methods of interest:

- `GreaterThanOrEqualZero(int x)` returning `x == 0 || x > 0`
- `IsGreaterThanOrEqualZero(int x)` using `bool b = ...; if (b) ...; return b;`
- `IsGreaterThanOrEqualZeroManual(int x)` using `x >= 0`

Continuation-only local helper used to refresh the dump later in the session:

- `/tmp/dotnet-jit-repro/bin/Release/net11.0/dotnet-jit-repro.dll`
- `/tmp/dotnet-jit-repro/Program.cs` now uses a tiny `OnTrue()` side-effect helper instead of `Console.WriteLine(...)`
- this preserves the same `bool local + if + return` CFG shape while avoiding extra console/tracing noise in the minimal repro

## What was tried

### 1. Retail runtime with `DOTNET_JitPath`

Tried using installed `dotnet` plus repo-built checked JIT through `DOTNET_JitPath`.

Result:

- no usable dump
- root cause: `JitPath` is effectively a debug-only load path in the VM

Relevant source:

- `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/vm/codeman.cpp:1796`
- `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/vm/codeman.cpp:2006`

### 2. Retail runtime with `DOTNET_AltJit`

Tried loading repo-built checked altjit through retail `coreclr`.

Result:

- failed with `mismatched JIT version identifier`

Conclusion:

- retail `coreclr` and repo-built checked JIT are not interface-compatible enough for this path

### 3. Build source-matched checked JIT

This succeeded:

```bash
./build.sh -subset clr.jit -c Checked -gcc -cmakeargs "-DCLR_CROSS_COMPONENTS_BUILD=1 -DFEATURE_EVENT_TRACE=0 -DFEATURE_EVENTSOURCE_XPLAT=0 -DFEATURE_PERFTRACING=0"
```

Important artifacts:

- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/bin/coreclr/linux.x64.Checked/libclrjit.so`
- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/bin/coreclr/linux.x64.Checked/libclrjit_unix_x64_x64.so`
- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/bin/coreclr/linux.x64.Checked/libjitinterface_x64.so`

### 4. Build source-matched checked `corerun` and `libcoreclr.so`

The workable route was a fresh `build-runtime.sh` subdir with cross-components enabled, plus explicit undef of `CROSS_COMPILE` in compile flags.

Command used:

```bash
./src/coreclr/build-runtime.sh -x64 -checked gcc -os linux -ninja -subdir xclrhack -component runtime -component hosts -cmakeargs "-DCLR_CROSS_COMPONENTS_BUILD=1" -cmakeargs "-DCMAKE_C_FLAGS=-UCROSS_COMPILE" -cmakeargs "-DCMAKE_CXX_FLAGS=-UCROSS_COMPILE" -cmakeargs "-DCLR_DOTNET_RID=linux-x64" -cmakeargs "-DCLR_DOTNET_HOST_PATH=/home/dev/work-base-20260421/workspace/platform/runtime/.dotnet/dotnet" -cmakeargs "-DCDAC_BUILD_TOOL_BINARY_PATH=/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/tools/cdac-build-tool/cdac-build-tool.dll" -cmakeargs "-DFEATURE_DYNAMIC_CODE_COMPILED=1" -cmakeargs "-DFEATURE_EVENT_TRACE=0 -DFEATURE_EVENTSOURCE_XPLAT=0 -DFEATURE_PERFTRACING=0"
```

Important artifacts:

- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/obj/coreclr/linux.x64.Checked/xclrhack/hosts/corerun/corerun`
- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/obj/coreclr/linux.x64.Checked/xclrhack/jit/libclrjit.so`
- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/obj/coreclr/linux.x64.Checked/xclrhack/dlls/mscoree/coreclr/libcoreclr.so`
- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/obj/coreclr/linux.x64.Checked/xclrhack/dlls/mscordac/libmscordaccore.so`

### 5. Build real `System.Globalization.Native-Static`

After installing ICU headers, this succeeded:

```bash
cmake -S /home/dev/work-base-20260421/workspace/platform/runtime/src/native/libs/System.Globalization.Native -B /tmp/sgn-build2 -DLOCAL_BUILD=1 -DCLR_CMAKE_TARGET_UNIX=1 -DCLR_CMAKE_TARGET_LINUX=1
cmake --build /tmp/sgn-build2 --target System.Globalization.Native-Static -j4
```

Artifact:

- `/tmp/sgn-build2/libSystem.Globalization.Native.a`

This archive includes:

- `GlobalizationResolveDllImport`

### 6. Re-link `libcoreclr.so` against the real static globalization lib

Used a local alias name expected by the xclrhack build:

- `/tmp/xclibs2/libSystem.Globalization.Native-Static.a -> /tmp/sgn-build2/libSystem.Globalization.Native.a`

Then:

```bash
ninja -t clean libcoreclr.so
env LIBRARY_PATH=/tmp/xclibs2 ninja libcoreclr.so
```

This produced a source-matched checked `libcoreclr.so` that got past the earlier globalization entrypoint failure.

### 7. Final working dump path

To get a usable dump, the runtime had to be launched with:

- source-matched checked native runtime pieces in `/tmp/xclrroot11`
- bootstrap `11.0.0-preview` managed runtime payload copied into `/tmp/corelibmix`
- repo-built `System.Private.CoreLib.dll` swapped into `/tmp/corelibmix`
- ICU shared libs preloaded

Temp dirs used:

- `/tmp/xclrroot11`
- `/tmp/corelibmix`

Repo-built corelib used:

- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/bin/coreclr/linux.x64.Checked/IL/System.Private.CoreLib.dll`

Command that produced the usable dump:

```bash
env \
  LD_PRELOAD=/usr/lib64/libicuuc.so.77:/usr/lib64/libicui18n.so.77:/usr/lib64/libicudata.so.77 \
  LD_LIBRARY_PATH=/tmp/xclrroot11:/tmp/corelibmix \
  CORE_LIBRARIES=/tmp/corelibmix \
  DOTNET_TieredCompilation=0 \
  DOTNET_ReadyToRun=0 \
  DOTNET_JitDump='*IsGreaterThanOrEqualZero*' \
  DOTNET_JitDumpASCII=0 \
  DOTNET_JitStdOutFile=/tmp/dotnet-jit-repro/full-jitdump-mix.txt \
  /home/dev/work-base-20260421/workspace/platform/runtime/artifacts/obj/coreclr/linux.x64.Checked/xclrhack/hosts/corerun/corerun \
  --clr-path /tmp/xclrroot11 \
  /tmp/dotnet-jit-repro/bin/Release/net10.0/dotnet-jit-repro.dll
```

The process later hit an unrelated checked assert:

- `System.Diagnostics.Tracing.XplatEventLogger::<EventSource_GetClrConfig>... not registered using DllImportEntry macro`

But that happened after the target methods had already compiled and the dump file had been written.

## High-signal findings from the dump

### Fresh continuation run from existing local artifacts

Using the already-built checked `xclrhack` runtime pieces plus a minimal `/tmp/dotnet-jit-repro` console app, a fresh dump was regenerated later in the session:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt`
- `/tmp/dotnet-jit-repro/full-jitdump-repro-123291.txt`

The same checked assert still occurs:

- `System.Diagnostics.Tracing.XplatEventLogger::<EventSource_GetClrConfig>... not registered using DllImportEntry macro`

But in the refreshed run the dump file is written first, so the fresh line references below are usable.

### Direct `return x == 0 || x > 0` method from the fresh continuation dump

Method:

- `Program:GreaterThanOrEqualZero(int):bool`

Relevant fresh dump location:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:1`

`optOptimizeBools` result:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:1762`
- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:1769`
- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:1771`
- reports `optimized 3 BBJ_COND cases in 3 passes`

Key transform:

- the `EQ/GT` pair is folded to `GE`
- then `fgFoldCondToReturnBlock()` folds the conditional into a direct `RETURN`

Fresh final codegen:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:2498`

High-signal instructions:

- `mov eax, edi`
- `not eax`
- `shr eax, 31`

### Problematic method

Method:

- `Program:IsGreaterThanOrEqualZero(int):bool`

Relevant dump location:

- method start: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:2698`

`optOptimizeBools` result:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5104`
- reports `optimized 0 BBJ_COND cases in 1 passes`

Shape after optimize-bools:

- still split across multiple blocks
- still uses temp bool local `V02 tmp1`
- no fold to a single `GE`

Relevant excerpt:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5120`
- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5136`

Final codegen pattern:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:6680`

Fresh continuation dump references:

- later `If conversion` rewrite: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5171`

High-signal instructions:

- `test edi, edi`
- `setg al`
- `test edi, edi`
- `cmovne ebx, eax`
- `test ebx, ebx`

This matches the earlier disasm suspicion: the method is not being canonicalized to `x >= 0`.

Important nuance from the fresh dump:

- `optOptimizeBools()` still does nothing for this method
- after that, `If conversion` *does* simplify the initial `store tmp1 = 1` / `store tmp1 = x > 0` diamond into a single `STORE_LCL_VAR tmp1 = SELECT(...)`
- but that happens only after `Optimize bools`, and no later phase turns the resulting `SELECT(eq0, 1, gt0)` shape into `GE`

### Manual `x >= 0` method

Method:

- `Program:IsGreaterThanOrEqualZeroManual(int):bool`

Relevant dump location:

- method start: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:6913`

Shape before / through optimize-bools:

- already has `GE` between `arg0` and `0`

Relevant excerpt:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:8684`

Final codegen:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:9849`

Fresh continuation dump references:

- `GE` already present through optimize-bools: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:8684`

High-signal instructions:

- `mov ebx, edi`
- `not ebx`
- `shr ebx, 31`

This is noticeably cleaner than the broken bool-temp shape.

## Updated diagnosis from the fresh continuation run

The refreshed dump narrows the likely fix surface further:

- `optOptimizeBools()` first tries `fgFoldCondToReturnBlock()` on `BBJ_COND` blocks and otherwise only enters the multi-block bool fold path when the false successor is itself `BBJ_COND`.
- the problematic method reaches `Optimize bools` as a six-block shape with materialized temp stores:
  - `BB01`: branch on `x == 0`
  - `BB02`: `tmp1 = x > 0`
  - `BB03`: `tmp1 = 1`
  - `BB04`: branch on `tmp1 == 0`
  - `BB05`: side-effect call
  - `BB06`: `return tmp1`
- that shape is neither:
  - `BBJ_COND -> BBJ_RETURN(1/0)` for `fgFoldCondToReturnBlock()`
  - nor `BBJ_COND -> BBJ_COND` for `optOptimizeBoolsCondBlock()`
- `If conversion` later rewrites the first diamond into `tmp1 = SELECT(x == 0, 1, x > 0)`, but in the observed pipeline it runs after `Optimize bools`:
  - `Optimize bools`: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5101`
  - `If conversion`: `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt:5171`
- after that rewrite, no later phase canonicalizes the `SELECT` form down to `x >= 0`, so the final codegen still carries the redundant `test/setg/test/cmovne/test` sequence

## Conclusion

The full checked dump supports the narrower diagnosis:

- the optimization rule is present in the JIT,
- but the `bool temp + if + return` shape from the issue does not trigger that optimization path,
- so the likely fix area is not adding a brand new compare-fold rule, but extending the existing optimization to cover this shape.

## Suggested next step

When resuming investigation or preparing a fix:

1. Focus on one of two narrow fix directions:
   - extend `optOptimizeBools()` / `fgFoldCondToReturnBlock()` so it can see through the materialized-bool local shape before `If conversion`
   - or add a later canonicalization that recognizes the post-`If conversion` shape `SELECT(x == 0, 1, x > 0)` as `x >= 0`
2. Use the refreshed local dump when iterating:
   - `DOTNET_JitDump='*GreaterThanOrEqualZero*'`
   - `/tmp/dotnet-jit-repro/full-jitdump-all-123291.txt`
3. Keep the fix narrow and pair it with a regression test in:
   - `/home/dev/work-base-20260421/workspace/platform/runtime/src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`
4. A plain functional regression test will preserve correctness, but it will not by itself prove the codegen win; keep a local JitDump/JitDisasm check in the loop while validating the patch.
5. Use the spike documents as the decision input before choosing the implementation path:
   - `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-a-spike.md`
   - `/home/dev/work-base-20260421/workspace/platform/runtime/issue-123291-option-b-spike.md`

## Post-patch local proof

A later local implementation pass completed **Option A** in repo code:

- `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp`
- `/home/dev/work-base-20260421/workspace/platform/runtime/src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`

Important implementation note from the real IR:

- the materialized bool temp in the target shape is normalized to `int`, not kept as a physical `TYP_UBYTE`
- the matcher therefore had to key on `genActualType(lvaGetDesc(lclNum)->TypeGet()) == TYP_INT`; a strict `TYP_UBYTE` gate did not match the real repro

### Checked post-patch JitDump

Fresh repro scaffold was recreated in `/tmp`:

- `/tmp/dotnet-jit-repro/Program.cs`
- `/tmp/dotnet-jit-repro/dotnet-jit-repro.csproj`
- `/tmp/dotnet-jit-repro/bin/Release/net11.0/dotnet-jit-repro.dll`

Authoritative post-patch dump:

- `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt`

Repro build command:

```bash
/home/dev/work-base-20260421/workspace/platform/runtime/.dotnet/dotnet build /tmp/dotnet-jit-repro/dotnet-jit-repro.csproj -c Release
```

Checked dump command:

```bash
env \
  LD_PRELOAD=/lib64/libicuuc.so.77:/lib64/libicui18n.so.77:/lib64/libicudata.so.77 \
  LD_LIBRARY_PATH=/tmp/xclrroot-proof:/tmp/checked-testhost \
  CORE_LIBRARIES=/tmp/checked-testhost \
  DOTNET_TieredCompilation=0 \
  COMPlus_TieredCompilation=0 \
  DOTNET_ReadyToRun=0 \
  COMPlus_ReadyToRun=0 \
  DOTNET_JitDump='*IsGreaterThanOrEqualZero*' \
  COMPlus_JitDump='*IsGreaterThanOrEqualZero*' \
  DOTNET_JitDumpASCII=0 \
  COMPlus_JitDumpASCII=0 \
  DOTNET_JitStdOutFile=/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt \
  COMPlus_JitStdOutFile=/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt \
  /tmp/xclrroot-proof/corerun \
  --clr-path /tmp/xclrroot-proof \
  /tmp/dotnet-jit-repro/bin/Release/net11.0/dotnet-jit-repro.dll
```

Where `/tmp/xclrroot-proof` was assembled from the checked `xclrhack` artifacts plus checked `System.Private.CoreLib.dll`, and `/tmp/checked-testhost` held the checked managed payload used earlier for subset test execution.

High-signal post-patch dump anchors:

- target method compile start:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:10074`
- `Optimize bools` for the target method:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12477`
- local fold marker:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12509`
- `optimized 1 BBJ_COND cases in 2 passes`:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12512`
- explicit `STORE_LCL_VAR tmp1 = GE(arg0, 0)` in `BB01`:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12530`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:12531`
- final 20-byte codegen for `Program:IsGreaterThanOrEqualZero(int):bool`:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13776`
- high-signal final instructions:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13786`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13787`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13788`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13789`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13793`
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:13797`
- manual `x >= 0` baseline still remains a 20-byte reference shape:
  - `/tmp/dotnet-jit-repro/full-jitdump-all-123291-after.txt:16937`

The important semantic change is now explicit in IR:

- the materialized-bool family reaches `Optimize bools`
- `fgOptimizeMaterializedBoolCompare` collapses the first diamond
- the target method contains an explicit `GE` local store before later phases
- final code size for the target method now matches the manual baseline size

### Checked `OptimizeBools` subset build and execution

The repo-native subset build was completed through the official test script, not via ad hoc project-only commands:

```bash
./src/tests/build.sh x64 checked -dir:JIT/opt/OptimizeBools /p:LibrariesConfiguration=Release
```

Important official build log anchors:

- `optboolsreturn` built:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/log/TestBuild.linux.x64.Checked.log:104`
- `Runtime_123621` built:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/log/TestBuild.linux.x64.Checked.log:105`

The official `run.sh --tree=JIT/opt/OptimizeBools` path did not find runnable subset markers for this build shape:

- `/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/log/TestRunResults_linux_x64_Checked.err:1`

Because of that, the checked execution proof was completed one layer lower using the already-built checked test outputs, checked `corerun`, and the checked `xunit.console.dll` runner:

```bash
env \
  CORE_LIBRARIES=/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/tests/coreclr/linux.x64.Checked/JIT/opt/OptimizeBools/optboolsreturn \
  LD_LIBRARY_PATH=/tmp/checked-testhost \
  LD_PRELOAD=/lib64/libicuuc.so.77:/lib64/libicui18n.so.77:/lib64/libicudata.so.77 \
  /tmp/checked-testhost/corerun \
  /tmp/checked-testhost/xunit.console.dll \
  optboolsreturn.dll \
  -xml /tmp/optboolsreturn.testResults.xml
```

and

```bash
env \
  CORE_LIBRARIES=/home/dev/work-base-20260421/workspace/platform/runtime/artifacts/tests/coreclr/linux.x64.Checked/JIT/opt/OptimizeBools/Runtime_123621 \
  LD_LIBRARY_PATH=/tmp/checked-testhost \
  LD_PRELOAD=/lib64/libicuuc.so.77:/lib64/libicui18n.so.77:/lib64/libicudata.so.77 \
  /tmp/checked-testhost/corerun \
  /tmp/checked-testhost/xunit.console.dll \
  Runtime_123621.dll \
  -xml /tmp/Runtime_123621.testResults.xml
```

Checked execution proof artifacts:

- `/tmp/optboolsreturn.testResults.xml`
- `/tmp/Runtime_123621.testResults.xml`

High-signal XML anchors:

- `optboolsreturn` assembly result:
  - `/tmp/optboolsreturn.testResults.xml:3`
- `optboolsreturn` test case:
  - `/tmp/optboolsreturn.testResults.xml:6`
- `Runtime_123621` assembly result:
  - `/tmp/Runtime_123621.testResults.xml:3`
- `Runtime_123621` test case:
  - `/tmp/Runtime_123621.testResults.xml:6`

Net result of the later local proof pass:

- the checked post-patch dump now shows the explicit `GE` fold for the target materialized-bool method
- the official `OptimizeBools` subset builds successfully through `src/tests/build.sh`
- both checked subset tests pass under checked runtime execution

## Notes

- No repo source code was modified during the original pre-patch investigation phase.
- A later local implementation/proof pass did modify:
  - `/home/dev/work-base-20260421/workspace/platform/runtime/src/coreclr/jit/optimizebools.cpp`
  - `/home/dev/work-base-20260421/workspace/platform/runtime/src/tests/JIT/opt/OptimizeBools/optboolsreturn.cs`
- The only new persistent repo file produced by the original investigation phase was this note; later proof artifacts remained under `/tmp`.
- Most runtime artifacts and temp directories under `/tmp` are investigation scaffolding and may need to be recreated later.
- The refreshed `/tmp/dotnet-jit-repro` helper is temporary scaffolding only; it is not part of the repo and was used just to regenerate current dump evidence from existing local checked artifacts.
