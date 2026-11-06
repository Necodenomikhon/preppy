# Implementation Plan: ETC1S encoder float32 bug: patch the encoder, triage and rebuild the catalog

**Track ID:** etc1s-clevel_20261102
**Spec:** [spec.md](./spec.md)
**Created:** 2026-11-02
**Status:** [~] In Progress

## Overview
Phases 1–2 shipped a mitigation (`--clevel 2`) and a texture gate before the root cause
was known. The root cause is now known (Phase 3, *Root cause*): a float32 accumulation
bug in the basis_universal ETC1S encoder that KTX-Software vendors. Phases 4–5 replace
the mitigation with an actual fix: preppy's container builds KTX-Software from source
with a one-line patch. That is temporary, until a KTX-Software release vendors a fixed
basis_universal. Once the fix is verified and the catalog triage is done, the gate is
no longer needed and Phase 6 removes it.

**Order:**
1. Task 3.1 (catalog triage, run to completion, with every flag verified as this bug);
2. Phase 4 (verify the patched encoder);
3. Phase 5 (container);
4. Tasks 3.2–3.4 and 3.6 (release, rebuild, timing; these use the patched container);
5. the Phase 3 verification (a re-triage, which uses the gate code);
6. Phase 6 (remove the gate).

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
- [x] Task 3.1: Run the gate over every published variant GLB; list affected
      objects/variants. *Done 2026-11-06.*
      - **Coverage:** all 13,224 published GLBs (2,204 objects × 6 variants) triaged,
        0 errors. 2,097 in WSL, the remaining 11,127 natively on Windows at
        below-normal priority (identical per-level results on PHerc1166Cr01). Each
        texture's decoded level 0 stands in as the source, since the published GLBs
        don't ship theirs.
      - **Flagged:** 859 GLBs, 963 textures, 750 objects. By GLB: FE2 398 / 2,204, PGS
        IR940 387 / 2,204, FE4 73 / 2,204, RGB 1 / 2,204, FE1 and FE3 0.
      - **Every flag checked against the root-cause signature** (bad blocks flat, darker,
        on an endpoint that can't reach the true brightness):
        - *Pass 1, the flagged level:* 382 textures matched.
        - *Pass 2, level 0:* when level 0 itself is the corrupted level, the triage sees
          the correct lower level as "brighter than the source". The test compares
          level 0 against the flagged level, restricted to areas flat in both. It was
          verified on PHerc0147Cr01 (97% unreachable, one code: base 148 / table 3,
          190 where the surface is 231) against a clean control (3 blocks of 4.2M).
          530 textures matched.

        | Variant | Bug at flagged level | Bug at level 0 | Unexplained |
        |---|---|---|---|
        | IR 1050 FE2 | 153 | 245 | 0 |
        | IR 1050 FE4 | 50 | 23 | 0 |
        | PGS IR940 | 179 | 262 | 50 |
        | RGB | 0 | 0 | 1 |
      - **Confirmed: 912 textures in 723 objects** (FE2 in 398, FE4 in 73, PGS IR940 in
        353). Per-texture results: [triage-flagged.csv](./triage-flagged.csv).
      - **Unexplained (51):**
        - 50 PGS IR940 textures with only 41–660 bad blocks (median 71), at L5 (256²)
          or L4 (512²). 37/50 are ≥70% darker on unreachable endpoints, but too few
          blocks to separate a small instance of the bug from ordinary error. Level 0
          is clean in all of them.
        - 1 RGB flag at exactly 1.00% with no flat or unreachable blocks: noise.
        - No evidence of a different defect.
      - **No margin at the gate threshold:** the strongest passing texture is 1.00%, and
        so is the weakest confirmed bug. 170 confirmed textures sit between 1% and 1.5%,
        288 under 2%. The distribution runs through the threshold, so real instances
        below 1% passed. Together with level-0 defects that only show when lower levels
        disagree, **every count above is a lower bound.**
      - **Rebuild scope (for Task 3.3):** rebuild **the whole catalog** with the patched
        encoder, not just the 750 flagged objects. A flagged-only rebuild leaves the
        unflagged sub-threshold and level-0 cases in place, and the Phase 5 patch marker
        renames every asset anyway.
      - **Tooling (investigator's workstation):** `E:\Wise\temp\preppy-triage\`, with
        results in `triage.jsonl`, `signature.jsonl` and `signature_l0.jsonl`, and the
        scripts and Windows toolchain under `win\`.

- [ ] Task 3.2: Cut a preppy release with Phases 1–2; confirm the GHCR image and the
      cluster `preppy*.sif` carry it. *Now waits for Phase 5:* the release must carry
      the patched encoder (and the effort level chosen in Task 4.5).
- [ ] Task 3.3: Re-run `pipeline_webify` for each affected object; republish. *Waits for
      Phase 5 / Task 3.2*, so the rebuild uses the patched encoder.
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
      cluster node against webify's `--time=1:00:00` Slurm limit. *Revised:* time it
      with the patched container from Phase 5, at the effort level chosen in Task 4.5;
      the gate is removed in Phase 6, so time it both with and without the gate.

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
chain is about 89M pixels) push the bad average lower. The real fix is the patched
encoder (Phases 4–5), until upstream fixes it.

### Verification
- [ ] A re-triage of the published catalog finds no flagged variants. *Runs after Task
      3.3 and before Phase 6* (it uses the gate code).

## Phase 4: Verify the patched encoder (Option 1)
Prove the one-line fix in the real tool before changing the container: KTX-Software
v5.0.0-rc2 built from source with the patch, run through preppy's own encode.

### Tasks
- [ ] Task 4.1: Write the patch against KTX-Software v5.0.0-rc2's vendored
      `external/basis_universal/encoder/basisu_etc.cpp`. In `etc1_optimizer::init`,
      accumulate the cluster average in `double` instead of `vec3F` (the change proven
      in Phase 3, *Root cause*). Keep it in the repo, e.g.
      `docker/patches/ktx-software/0001-basisu-etc1-optimizer-double-average.patch`,
      with a header naming the bug, preppy#5, and the condition for dropping it.
- [ ] Task 4.2: Build KTX-Software v5.0.0-rc2 from source with the patch on Linux x86_64,
      matching the release `.deb`'s configuration where it matters (Release, the same
      basis_universal options and SSE setting). Record the CMake flags, build time and
      toolchain. Confirm `ktx --version` and that an unpatched build from the same
      source reproduces the release `.deb`'s FE2 output byte for byte, so build
      differences can't masquerade as the fix.
- [ ] Task 4.3: Re-encode PHerc1166Cr01 FE2 from its archived `material_0.png` with the
      patched `ktx`, at the old default (`--clevel 1`) and at `--clevel 2`, via
      preppy's `encode_ktx2()`. Both must pass the texture gate at every level (the
      instrumented build predicts 0.00%).
- [ ] Task 4.4: Regression check, patched vs unpatched, on:
      - textures that are already clean: PHerc1166Cr01 FE1/FE3/FE4/RGB/PGS, with
        decoded level 0 as the stand-in source;
      - flagged textures from the **completed** Task 3.1 triage, the same way: all of
        them if time allows, otherwise a sample covering every affected variant and
        the full range of worst-level shares.

      Record per texture: bad-block share per level, size, encode time, and whether the
      output changed at all. Every flagged sample must come out clean.
- [ ] Task 4.5: Decide the ETC1S effort level for patched builds: keep `--clevel 2`, or
      return to the default 1 (about half the encode time, ~2–3.5% smaller). Decide
      from Task 4.4's numbers and record the reason. Changing it is a codec-option
      change, so it renames assets via the hash (Task 1.3).

### Verification
- [ ] With the patched `ktx`, FE2 and every sampled flagged texture pass the gate at the
      chosen effort level; the unpatched control reproduces the release output.

## Phase 5: Patched encoder in the container
Ship the verified build in the image webify uses, and make patched outputs
distinguishable from unpatched ones.

### Tasks
- [ ] Task 5.1: Dockerfile: build KTX-Software v5.0.0-rc2 from source with the patch,
      instead of installing the release `.deb`. Use a multi-stage build so build
      dependencies stay out of the final image, keeping both amd64 and arm64
      (`build_docker.yml` builds multi-arch). Record the image build-time increase.
- [ ] Task 5.2: `singularity/preppy.def`: the same, so HPC builds match the Docker image.
- [ ] Task 5.3: Tests first, then mark patched outputs in the asset hash. `ktx
      --version` is the same for patched and unpatched builds, so `tool_versions`
      won't change. Have the image declare its patches (e.g. an env var or a file
      listing patch ids), have preppy read it, and fold it into the hash config.
      Assets encoded by the patched build then get new `immutable` names.
- [ ] Task 5.4: Decide preppy's behaviour on an unpatched `ktx` (e.g. a `brew install ktx`
      outside the container), since Phase 6 removes the gate that currently catches
      the defect there. Options: warn, refuse ETC1S, or fall back to UASTC. Implement
      the choice, tests first, and document it in the README.
- [ ] Task 5.5: Docs:
      - ADR 0002 amendment: the patched encoder, why, and when to drop it, namely a
        KTX-Software release vendoring a fixed basis_universal;
      - `tech-stack.md` and README install notes (container vs. unpatched installs);
      - `CLAUDE.md`.
- [ ] Task 5.6: CI: confirm `build_docker.yml` still builds both architectures within
      its limits, and run the manual integration job against the new image.

### Verification
- [ ] An image built from the branch re-encodes FE2 clean at the chosen effort level;
      `python -m pytest tests/` green; the hash test shows patched and unpatched
      outputs get different names.

## Phase 6: Remove the texture gate
*Decision (2026-11-04):* with the encoder fixed, the gate is no longer needed. Remove
it only after the catalog triage (Task 3.1), the patched-encoder verification (Phases
4–5) and the post-rebuild re-triage (Phase 3 verification) are done, since the
triage tooling uses the gate's code.

### Tasks
- [ ] Task 6.1: Remove the gate from preppy:
      - `validate_ktx2`, `check_levels`, `block_defect_fraction`, `TextureCheckResult`,
        `reference_ktx2_cmd`, `extract_levels_cmd`, `_decode_levels`,
        `_ktx2_dimensions` and the gate constants in `texture.py`;
      - the `file_prep` wiring and the `--texture-check` / `--on-texture-defect` flags;
      - their tests.

      `ktx2_codec_args` stays (the hash uses it). The gate isn't in the hash, so asset
      names don't change.
- [ ] Task 6.2: Docs: remove the gate from the README flag table, `CLAUDE.md` and ADR 0002.
      Record in the ADR that it was removed once the encoder fix was verified, and that
      it remains in git history if it's needed again (e.g. on a KTX-Software bump).
- [ ] Task 6.3: Record the final triage results (affected objects, rebuild list,
      re-triage outcome) in this plan before the triage tooling's dependency is gone.

### Verification
- [ ] `python -m pytest tests/` green; `preppy -h` lists no texture-check flags; a
      re-encode of FE2 with the patched container is unchanged by the removal
      (byte-identical KTX2).

## Final Verification
- [ ] All acceptance criteria met
- [ ] Tests passing
- [ ] Documentation updated
- [ ] Ready for review

---

_Generated by Conductor. Tasks will be marked [~] in progress and [x] complete._
