# Implementation Plan: ETC1S encode quality (default --clevel corrupts grayscale textures)

**Track ID:** etc1s-clevel_20261102
**Spec:** [spec.md](./spec.md)
**Created:** 2026-11-02
**Status:** [~] In Progress

## Overview
Phase 1 is the minimal fix and can merge on its own. Phase 2 adds the guard that would
have caught it. Phase 3 is operational: find and rebuild what already shipped.

## Phase 1: Encode fix + cache correctness (mergeable alone)

### Tasks
- [x] Task 1.1: Tests first. `test_encode_ktx2_cmd_etc1s_default` asserts `--clevel 2`;
      the UASTC test asserts no `--clevel`.
- [x] Task 1.2: Add `--clevel 2` to the ETC1S branch of `encode_ktx2_cmd`. Done as a
      mode -> codec-options table (`_KTX_CODEC_ARGS`) behind `ktx2_codec_args()`, so
      Task 1.3 can hash exactly the options the encode uses.
- [x] Task 1.3: Test first, then fold the encode parameters into the `file_prep` hash
      config (e.g. the codec's extra argv), so a flag change gives a new asset name.
- [x] Task 1.4: Re-encode PHerc1166Cr01 FE2 from its archived `material_0.png` through
      preppy. Confirm the per-level check passes and the hashed name differs from
      `aa9e04c1`. *Done for the encode:* `texture.encode_ktx2()` with KTX 5.0.0-rc2 gives
      0.00% / 0.06% bad blocks at L2 / L3 (shipped: 11.47% / 14.28%), byte-identical
      to the hand-run `--clevel 2` encode. *Rename on the real asset* needs a full run
      with the source OBJ; the rename logic is unit-tested (Task 1.3) and the real
      asset is checked in Task 3.4.
- [x] Task 1.5: Records. Amend ADR 0002 and `tech-stack.md` (ETC1S at `--clevel 2`,
      with the reason), and add forward pointers to this track in `spike/FINDINGS.md`
      (KTX2 mode decision) and harden-migrate Task 1.2.
- [-] Task 1.6: *Moved to Task 3.6* (needs cluster access; the person running webify
      takes it with the rebuild).

### Verification
- [x] `python -m pytest tests/` green (110 passed, 10 skipped); the FE2 re-encode is
      clean. Rename on the real asset carried to Task 3.4.

## Phase 2: Texture quality gate

### Tasks
- [x] Task 2.1: Decide the decoder (Node + Basis transcoder vs a Python binding) and the
      metric/threshold; record the choice in the plan. **Decision:**
      - *Decoder:* `ktx extract --transcode rgb8 --level all` (the `ktx` preppy already
        requires; no new dependency). About 8 s for an 8K chain.
      - *Reference:* the source PNG through the same `ktx create --generate-mipmap`
        without `--encode`, extracted. Same mip filter, so only compression error
        remains. (A box-downsampled reference false-flagged clean high-detail
        textures, up to 11.7% at L7 on PGS IR940, from the filter difference alone.)
      - *Metric:* per level, the share of 4×4 blocks whose mean differs from the
        reference by more than 25 (8-bit) in any channel. Fail if any level whose short
        side is at least 256 px exceeds 1%.
      - *Evidence:* defective encodes 7.8–17.8% at the worst checked level (shipped FE2,
        stand-in default/qlevel/maxep); clean encodes at most 0.22% (FE2 `--clevel 2`
        from the real source, stand-in `--clevel 2/3/5` and UASTC, RGB and PGS IR940 at
        `--clevel 2`). Levels below 256 px are skipped: on RGB, ordinary ETC1S colour
        error reached 1.82% at 127×95.
      - *Defaults:* on by default (it needs only `ktx`); `--on-texture-defect fail|warn`,
        default `fail`, mirroring `--on-over-budget`. Check intermediates go in a
        deleted temp dir so `--keep-tmp` archives don't grow. The check doesn't change
        output, so it stays out of the hash.
- [x] Task 2.2: Tests first. A fixture pair: the shipped FE2 `.ktx2` (must flag) and the
      `--clevel 2` encode (must pass), or small synthetic equivalents if 8K fixtures are
      too heavy for the repo. *Done synthetically* (the 8K files are 4–30 MB each):
      command builders, block metric, per-level gate, orchestrator fail/warn/off, CLI
      defaults, and a `ktx`-backed clean-encode test. The real FE2 pair was checked
      by hand in Task 2.3.
- [x] Task 2.3: Implement `texture.validate_ktx2()` and wire it into `file_prep` after
      encode, with `--on-texture-defect {warn,fail}` mirroring `--on-over-budget`.
      *Cost:* about 35 s per 8K texture (46 s at 8192²), almost all of it building
      the uncompressed reference chain (~29 s of single-threaded mip generation;
      `--threads` doesn't help). Raw extraction instead of PNG saved ~15 s.
      *Possible optimization if Task 3.6 finds the time tight:* encode in two steps,
      uncompressed `ktx create --generate-mipmap` then `ktx encode --codec basis-lz
      --clevel 2`, and reuse the first output as the gate's reference. Measured on FE2:
      pixel-identical at all 13 levels and the same size (only `KTXwriter` differs),
      saving ~25 s per texture. It needs the 200 MB intermediate deleted before
      `--keep-tmp` archives it.
- [x] Task 2.4: README/CLAUDE.md: document the gate alongside the Hausdorff gate.

### Verification
- [x] The gate flags the archived FE2 `.ktx2`, passes the `--clevel 2` encode, and passes
      PHerc1116 Scorza FE2 (the negative control). *Run through `validate_ktx2()` with
      KTX 5.0.0-rc2:* shipped FE2 FAIL (L3 14.20%, L2 11.46%); FE2 `--clevel 2` PASS
      (all 0.00%); stand-in default FAIL (L3 17.75%, L0 8.77%). *Negative controls run
      instead of Scorza* (its archive was no longer available): PHerc1166Cr01 RGB and
      PGS IR940 at `--clevel 2`, both PASS (worst 0.22% / 0.15%). Suite: 120 passed /
      11 skipped without `ktx`; 121 / 10 with it. The full-pipeline integration tests
      (need ImageMagick + gltfpack) were not run.

## Phase 3: Catalog triage + rebuild (operational)

### Tasks
- [ ] Task 3.1: Run the gate over every published variant GLB; list affected
      objects/variants.
- [ ] Task 3.2: Cut a preppy release with Phases 1–2; confirm the GHCR image and the
      cluster `preppy*.sif` carry it.
- [ ] Task 3.3: Re-run `pipeline_webify` for each affected object; republish.
- [ ] Task 3.4: Confirm the live PHerc1166Cr01 FE2 renders correctly in the viewer and its
      hashed name is no longer `aa9e04c1` (carried over from Task 1.4); close
      dri-voyager#13 and preppy#5.
- [x] Task 3.5: Resolve the remaining open questions: rc1 behaviour (test with the rc1
      `.deb`) and the upstream report (file it, or record why not).
      - *rc1:* KTX 5.0.0-rc1 (`v5.0.0-rc1~5`, the spike's version) re-encoding the
        archived FE2 PNG at the old default gives the same per-level result as rc2
        (L2 11.47%, L3 14.28%). The defect predates the rc2 pin; the spike's
        toolchain had it too.
      - *Upstream (corrected 2026-11-04):* an earlier note here called this documented
        `-comp_level` behaviour, not a bug. That was wrong; see **Root cause** below.
        It is a basis_universal bug, still present on its master, to be reported.
- [ ] Task 3.6: *(from 1.6; needs cluster access)* Time a full 6-variant preppy run on a
      cluster node against webify's `--time=1:00:00` Slurm limit, with ETC1S at
      `--clevel 2` and the texture gate on.

### Root cause (identified 2026-11-04)
Proven with an instrumented build of the basis_universal encoder that KTX-Software
v5.0.0-rc2 vendors (lib version 210), driven exactly as `lib/src/basis_encode.cpp`
drives it. Its output at `--clevel 1` reproduces the shipped FE2 numbers exactly
(L2 11.46%, L3 14.20%).
1. ETC1S shares one endpoint codebook across every mip level (KTX passes each level
   as a separate source image). FE2's near-white subblocks form one huge cluster:
   3,603,870 subblocks, 98.5% pure white, 28.8M pixels.
2. `etc1_optimizer::init` (basisu_etc.cpp) sums the cluster's colours in **float32**
   (`vec3F avg_color; avg_color += fc`). Past 2^32 the float spacing exceeds 510, so
   adding 255 no longer changes the sum: it sticks at exactly 4,294,967,296. The
   average comes out 148.97 instead of 254.95.
3. At `--clevel 1` the cluster optimizer searches at Medium quality around that wrong
   average and returns base 156 / intensity table 1, which can't exceed 173. At
   `--clevel 2` (Slow) the broader search escapes to base 222 / table 5. **Same bug,
   different luck:** `--clevel 2` is a workaround, not a fix.
4. `refine_endpoint_clusterization` only moves a block to a cluster with an intensity
   table no larger than its current one, within its parent cluster. Table 1 leaves no
   white-capable target, so the white blocks stay. (FE2 ×0.98 hit the same bug, average
   152.89 instead of 249.99, but its endpoint landed on table 4 and refinement rescued
   it.)
5. The backend's codebook refit (`reoptimize_remapped_endpoints`) runs only at level
   > 1. At level ≤1 endpoint RDO repairs blocks in raster order by trying endpoints
   near the previous block's, restricted to `COLOR_DELTA_THRESH = 8` from the
   current endpoint. Runs that start on the grey endpoint can't jump to white
   (difference 36), so they persist rightward and downward: the staircase regions.
   With `--no-endpoint-rdo` nothing is repaired (~40% of blocks bad at every level).

**Proof:** changing only that accumulator to `double` and re-encoding at the old
default `--clevel 1` gives base 255 / table 0 for the cluster and **0.00% bad blocks
at every level**. basis_universal master still has the float32 accumulator.
Consequence: `--clevel 2` is a mitigation, and larger or brighter clusters (an 8192²
chain is about 89M pixels) push the bad average lower. The texture gate remains the
guarantee; the real fix is upstream.

### Verification
- [ ] A re-triage of the published catalog finds no flagged variants.

## Final Verification
- [ ] All acceptance criteria met
- [ ] Tests passing
- [ ] Documentation updated
- [ ] Ready for review

---

_Generated by Conductor. Tasks will be marked [~] in progress and [x] complete._
