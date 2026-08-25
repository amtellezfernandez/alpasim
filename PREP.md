# Upstream PR prep — verified, awaiting OK to open

Prepared 2026-08-13 against `NVlabs/alpasim` `main` @ `1e801ca` (tip re-checked at prep
time) and `NVIDIA/flashdreams` @ `ac214dd`. Every branch is authored and committed as
`amtellezfernandez <177202985+amtellezfernandez@users.noreply.github.com>`. Nothing has
been pushed or opened. **Before any `gh` action: `gh auth switch --user amtellezfernandez`.**

Open upstream: #127 (runtime fallback dynamics), #129 (ARM64 docker build),
#147 (FlashDreams tag docs), #148 (CameraFrame contract — reopened with receipts),
#149 (rollout seeds via RolloutSpec — the #128 follow-up, maintainer-invited),
#150 (issue: route-generator plugin appetite check — branch `pr/route-generator-plugin`
@ `be1141b` ready if yes; opened 2026-08-13).

#149 lifecycle note: built BEFORE replying to #128 so the 16-day-late answer arrived as a
delivery, not a promise. Branch `pr/rollout-seed`, template = start_time_offset_us's exact
path; derivation base+k mirrors upstream's own inference_seed convention. Verified on the
Spark: protos regenerated, targeted suites 30/30, full runtime 493 passing; 3 test_with_mocks
failures isolated to the containerized proto-regen harness via pristine-source control run
(fail identically without the change), stated as such in the commit. Reply on #128 posted
same moment, pointing at the PR.

## AlpaSim candidates — branches in THIS clone (`~/sota/alpasim-prs`)

Every pair stacks cleanly, and the driver-touching pair stacks on top of open #127
(verified with `git am` on every combination).

| # | branch | verification performed | risk |
|---|--------|------------------------|------|
| 1 | `pr/targetarch-stage-scope` | **PARKED — do not open standalone.** Audited 2026-08-13: on main, `TARGETARCH` is consumed nowhere inside the stage (`git grep` shows only the ARG and the FROM), so standalone this is dead code fixing a bug that cannot fire — the exact thing a reviewer rejects. The scoping fact itself is verified (repro: `PROBE=[]` vs `PROBE=[arm64]`) and the fix already lives inside #129, where its consumer exists. Open only if a maintainer asks for the split (#129's comment offers it conditionally). Fork branch deleted; local branch kept. |
| 2 | `pr/lazy-model-imports` | **PARKED — no utility from upstream's chair.** vam/alpamayo are *base* deps of alpasim_driver (correct installs import fine), and upstream's own transfuser plugin imports via `models.base` directly — the pattern already exists. Failure only bites partial envs like ours; AlpaBridge carries the override, which is where the utility lives. |
| 3 | `pr/utils-rs-negative-zero-serialization` | **HOLD — real divergence, no victim found.** Measured against python protobuf (skips -0.0; Rust encoder emits it), but the only byte-hashing path upstream (`force_gt_cache_signature`) uses python's own serializer, self-consistent. No observable failure today → fails the necessity bar. Revisit if a byte-comparison path crossing both encoders appears, or as a rider on a utils_rs change. |
| 4 | `pr/cameraframe-type-mismatch` | **OPEN, [#148](https://github.com/NVlabs/alpasim/pull/148)**, body now self-contained. Audit 2026-08-13 of the necessity claim: (a) the proof needs no AlpaBridge — upstream's own `CameraFrame` NamedTuple vs the tuple its `_prepare_camera_images` builds gives `isinstance -> False` and `AttributeError`, verified by faithful replication of both declarations; (b) entry-point path confirmed: AlpaBridge registers 5 models under `alpasim.models`, upstream feeds them those tuples in-process; (c) **corrected a misread**: `vavam_model.py:58`'s bare `frames[-1].image` is NOT evidence — vavam runs the external-driver path where AlpaBridge builds its own `DriverCameraFrame` dataclass; (d) **nuance measured**: removing AlpaBridge's *positional fallback* alone degrades silently to `None` (freshness check disabled), not a crash — the `AttributeError` belongs to the docs-faithful reader. Body leads with the upstream-only proof; AlpaBridge is one sentence of cost evidence with permalinks, optional reading. |
| 5 | `pr/session-event-idempotency` | **HOLD until #127 resolves — and explicitly NOT to be folded into #127** (decided 2026-08-13): #127's thesis is removing driver-side logic per the reviewer's preference, its evidence is unit-test red/green while #5's is production rollouts, and the bugs are unrelated (dynamics vs session races). Bundling would contradict #127's own concession and price it at the weaker evidence. Trigger to offer #5: the reviewer asking about other robustness issues in that thread. |
| 6 | `pr/route-generator-plugin` | **ISSUE-FIRST, not a PR.** 6-file feature in a repo where external PRs sit for months; ask appetite in a short issue (the #128 thread shows the maintainer inviting exactly this pattern). Branch + 26-test verification stay ready for a yes. Issue draft ready (scratchpad `i_routegen.md`, 2026-08-13), reframed per the user: anchored on upstream's own declaration (PLUGIN_SYSTEM.md line 3, "without modifying the core codebase"), NOT on the "four registries exist" analogy; explicitly asks whether routes being absent from the documented models/configs/tools list is a deliberate boundary. Pre-open investigation (2026-08-13): no prior issue/PR asks for this (searched "route"/"plugin", all states); #138 closed benign (usage explanation); tip unchanged at 1e801ca; all draft claims verified against main (ABC line 18, closed dispatch in create(), doc quote verbatim). **Branch defects FIXED (2026-08-13, commit `be1141b`, now 9 files / +62):** (a) the alpasim_plugins import is now deferred inside create()'s plugin path per the constraint main's own `src/runtime/tests/conftest.py` documents (CI runs runtime tests without alpasim_plugins); the invented `from_context` contract was replaced with upstream's `PluginRegistry.create(name, **kwargs)` and the kwargs contract is documented in the docstring; (b) plugin-path test added using their `patch_plugin_registry` conftest helper (asserts group, name, kwargs identity, sentinel return with the enum path bypassed); (c) parity additions: `route_generators` exported in plugins `__init__.py` like the other four registries, label in `info.py`. Spark verification (container `alpasim-arm64-aug2026`, `~/rgtest`): runtime 22 passed vs pristine 21 (our test is the +1); 2 map-test failures and 2 plugins "driver_installed" failures are pristine-identical harness artifacts (control runs); ruff format clean, ruff check adds ZERO findings vs pristine (the 2 I001s are detection artifacts that fire on upstream's own untouched files in the real layout — proven on event_loop.py and on main's own test_route_generator.py; our top-level import block is byte-identical to main). Commit message rewritten to the PLUGIN_SYSTEM.md-contract framing (no registry-count analogy). NOTE: runtime and plugins test packages are both named `tests` — pytest them in separate invocations or collection fails. Context: that conftest fixture exists on main with ZERO runtime callsites → plugin-from-runtime wiring is likely already happening internally on GitLab; an "already coming internally" reply to the issue is a plausible and acceptable outcome. Same-file activity: Dingrui-Wang's open #143 rewrites RouteGeneratorMap internals (different region than create(); no textual conflict expected, but same file under active review). |
| 7 | `pr/video-model-docs-tag` | **OPENED as [#147](https://github.com/NVlabs/alpasim/pull/147)** (2026-08-13). Audit before opening: fires today on both current mains; FlashDreams' ARG default has never had another value (no version defense); direction forced (FlashDreams internally consistent). Diff slimmed to one line; escape hatch moved to the PR body. | lowest |

**Do not open:** `docker-local-extras` (ON HOLD — `uv sync --extra` installs `+cpu` torch on
aarch64, measured), `route-waypoints-in-prediction-input` (superseded upstream).

### Suggested order (revised 2026-08-13 after per-candidate utility audit)
#4 is the only remaining direct-open candidate (utility belongs to upstream's plugin
ecosystem). #5 waits on #127's outcome; #6 goes issue-first; #1/#2 parked; #3 held.

### To open (after OK), per branch
```bash
cd ~/sota/alpasim-prs && gh auth switch --user amtellezfernandez
git push fork pr/<name>:pr/<name>
gh pr create --repo NVlabs/alpasim --base main \
  --head amtellezfernandez:pr/<name> \
  --title "$(git log -1 --format=%s pr/<name>)" \
  --body-file <body>   # bodies: derive from third_party/alpasim_overrides/<name>.md in wayspan
```

## FlashDreams candidates — branches on the Spark (`~/flashdreams`)

No fork of `NVIDIA/flashdreams` exists yet — `gh repo fork NVIDIA/flashdreams --clone=false`
is the first step after OK, then push from the Spark clone (needs auth there) or bundle the
commits to the laptop and push via a fork remote.

| branch | verification performed | risk |
|--------|------------------------|------|
| `fd/apps-workspace-copy` (`e076124`) | The strongest verification in this set: with the pristine lockfile, the build fails 255-vs-256 on any arch; with this one line it builds end to end — the resulting image is the one running OmniDreams in production on GB10. | lowest — upstream is broken as shipped |
| `fd/te-aarch64-pin` (`f797635`, independent of the above) | `uv lock` resolves 256 packages; lock now carries `transformer_engine_cu13-2.17.1-...-manylinux_2_28_aarch64.whl`. Latent fix (dev extra only); not build-verified since the default build never installs it — say so in the PR. | low |

## Standing rule (added 2026-08-13, after #148)
If the open-time audit *weakens* the case — a victim disappears, a premise downgrades —
the finding goes back for a decision before anything is pushed or opened. An audit that
only ever green-lights is not an audit. Corollary on framing: observed-bug and
contract-hygiene are different genres of PR; never sell the second as the first, and
always state "no current consumer is affected" explicitly when it is true.

## Identity + hygiene checklist before opening anything
- `gh api user --jq .login` → must print `amtellezfernandez`
- committer email on every commit is the noreply (verified at prep)
- no private hostnames/paths in any commit message or body (verified at prep)
