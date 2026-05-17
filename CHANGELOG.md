# Changelog

All notable changes to RewardHarness are recorded here. Versions follow [SemVer](https://semver.org/). Dates are in ISO 8601 (UTC).

## [Unreleased]

Post-v0.1.2 polish, slated for v0.1.3:

### Added

- `scripts/check_links.sh` &mdash; audits every markdown link in the docs (relative paths always; external URLs with `--external`).
- `WALKTHROUGH.md` &mdash; 5-step Vertex AI service-account setup guide with direct GCP console deep-links; step 8 now shows the fast `python scripts/run_benchmark.py --config configs/default.yaml` paper-reproduction path against the shipped `src/library/` (no evolution required first).
- README release badge auto-updates from the latest GitHub tag.
- Per-author `affiliation` entries in the JSON-LD `ScholarlyArticle` block (21 author&ndash;org links).
- `examples/score_pair.py --show-chain` flag to print the full `<think>/<tool>/<obs>/<answer>` reasoning trace.
- `REWARDHARNESS_SUBAGENT_MODEL` env var lets you swap in a non-Qwen OpenAI-compatible Sub-Agent without editing source. Honoured by **every** vLLM call site — `SubAgent._call_vllm`, `Library.call_tool`, `Evolver._validate_tool_prompt`, plus the server-side launchers `serve_vllm_multi.sh`, `_launch_all_vllm.sh`, `sbatch_vllm.sh`, and `start_vllm_remote.sh` — so VLM-swap setups are coherent end-to-end.
- `CLAUDE_API_BASE_URL` / `CLAUDE_API_KEY` env vars in `vanilla/bench_{claude,genaibench,imagenhub}.py`. Previously the URL was hardcoded to a dead internal proxy, so the Claude baselines were not reproducible without editing source.
- `.env.example` now lists every env var the code reads (13 new entries) &mdash; `REWARDHARNESS_SUBAGENT_MODEL`, the `serve_vllm_multi.sh` knobs (`NUM_GPUS`, `ENDPOINTS_PER_GPU`, `BASE_PORT`, `GPU_MEM`, `MAX_MODEL_LEN`, `VLLM_MODEL_PATH`), BCM cluster overrides (`RH_SKIP_ENV_PIN`, `SLURM_PREFIX`, `CUDA_LIBS`), Gemini gateway pair, Claude proxy pair. WALKTHROUGH step 4 (`cp .env.example .env`) is now a one-stop discovery mechanism.

### Changed

- `CLAUDE.md` rewritten with an explicit "for AI coding agents" preamble; dropped the internal-only `a-tool/edit-reward/` reference; tightened the no-coauthor rule to also forbid AI-attribution footers.
- `Makefile` &mdash; `make benchmark` defaults to the paper-evolved `src/library/` (6 entries) instead of `examples/seed_library/` (3 entries). New users running `make benchmark` to verify paper headline numbers now see the actual paper accuracy instead of seed-library accuracy.
- README "Repository layout": `data/` row clarified (HuggingFace caches into `~/.cache/huggingface/`, not the repo's `data/` dir).
- README "Architecture": honest description of the evolution gate &mdash; replaces "kept only if held-out accuracy improves" with the actual `explore_margin` semantics (small dips permitted within tolerance).
- README "Key config" / `configs/default.yaml` &mdash; the `model:` block is now flagged INFORMATIONAL; no code reads those keys. Serving knobs are env vars consumed by `scripts/serve_vllm_multi.sh`.
- Footer on the website lists both code mirrors (TIGER-AI-Lab / KlingAIResearch).
- Website "Method" card now describes the actual phase A/B/C structure from `src/pipeline.py` instead of a fictitious "five stages" framing.
- Website figures now use `loading="lazy"` and `decoding="async"`, cutting first-paint bandwidth by ~820 KB for visitors who don't scroll past Abstract.
- Website "Try it yourself" callout now shows `--show-chain` (so the printed trace matches the section's headline example) and includes the no-image-needed fast path `python scripts/run_benchmark.py --config configs/default.yaml`.
- Website Table 2: Flux.1 Kontext [dev]'s 3.52 Overall is now marked second-best (matching the "tied at 3.52 with a smaller backbone" claim in the caption); previously RewardHarness's 3.52 was uniquely highlighted.
- Website mobile (≤735px): the dark-mode toggle stays visible (the previous CSS hid the whole `.nav-links` `<ul>`, taking the toggle with it).
- `vanilla/README.md` &mdash; backend/env-var table corrected: `gemini_bench_*.py` go through an OpenAI-compatible "Gemini gateway", not a direct Vertex AI client (their `--model` accepts Gemini or Claude ids — whatever your gateway supports).
- `score-guidelines/README.md` rewritten: the templates ARE loaded by `src/sub_agent.py` at inference time (the old text claimed otherwise), and the internal scoring scale is 1-4 end-to-end (the old text claimed a 1-5 remap).
- WALKTHROUGH "Where things live" table &mdash; corrected the VLM-swap recipe: split into OpenAI-compatible (export env var, no source edit) and non-OpenAI (subclass) paths, matching what's actually in `src/sub_agent.py`.

### Removed

- `vanilla/bench_wanqing.py` and hardcoded credentials from `vanilla/gemini_bench_*.py` (already shipped in v0.1.1's security patch; carrying the note here for completeness).
- Stale `data/checksums.txt` placeholder that no script ever populated.

### Fixed

- `scripts/run_evolution.py` now has the executable bit set, matching its siblings.
- `scripts/start_vllm_remote.sh` gained a header docstring explaining the SSH/Slurm invocation pattern, and now respects the same env-var conventions as `serve_vllm_multi.sh` (`VLLM_PYTHON`, `VLLM_MODEL_PATH`, `REWARDHARNESS_SUBAGENT_MODEL` via `--served-model-name`, `MAX_MODEL_LEN`).
- `scripts/reproduce.sh` step 4 now resets the `waited` counter per vLLM port (was a shared accumulator across all 16 ports, so the last few ports got near-zero timeout budget) and bumps per-port budget to 10 min for cold-start safety. **Also major fix:** step 4 reads `configs/endpoints.txt` (the same file the pipeline reads) instead of hardcoded ports 8000-8015, which mismatched `serve_vllm_multi.sh`'s 4-endpoint default and caused ~2 hours of timeout failure on default-config runs.
- `scripts/setup_env.sh` step labels harmonised &mdash; was `[1/3]`/`[2/3]`/`[3/3]`/`[4/5]`/`[5/5]`, now consistently `[N/5]`.
- `scripts/serve_vllm_multi.sh` &mdash; Bright Cluster Manager paths (`/cm/...`) are now overridable via `SLURM_PREFIX`, `CUDA_LIBS`, or skippable entirely with `RH_SKIP_ENV_PIN=1`. Previously broke `LD_LIBRARY_PATH` on vanilla Ubuntu/RHEL hosts.
- `scripts/download_data.sh` &mdash; dropped a dead checksum-generation step that wrote a `data/checksums.txt` no other script ever read. Replaced with a one-liner showing how to compare HuggingFace's built-in `_fingerprint` instead.
- `scripts/check_env.py` &mdash; vLLM endpoint probes now run in parallel (was sequential; with 16 endpoints all timing out, preflight took 48 s instead of ~3 s).
- `scripts/_launch_all_vllm.sh`, `scripts/sbatch_vllm.sh` &mdash; respect `NUM_GPUS` / `BASE_PORT` / `GPU_MEM` / `MAX_MODEL_LEN` / `VLLM_MODEL_PATH` env vars matching the public `serve_vllm_multi.sh`; previously hardcoded 4 GPUs and 0.85 GPU-mem. `sbatch_vllm.sh` also dropped two unsubstituted `/path/to/your/...` template paths in favour of a fail-fast `$RH_ROOT` requirement.
- `scripts/run_evolution_seed.sh` &mdash; generated YAML now uses `gemini:` (matching `configs/default.yaml`) instead of `claude:`, which would have `KeyError`'d at Router construction and also violated the no-Claude-API project rule.
- `scripts/run_all_benchmarks.sh` &mdash; summary table now shows `GenAI` and `average` columns (was only K=2/K=3/K=4), so the headline "which iter scored best on average" is visible at a glance. Inline-python invocation hardened (sys.argv instead of bash-substitution into source).
- `src/library/__init__.py::call_tool` and `src/evolver.py::_validate_tool_prompt` &mdash; tool dispatches and Phase-B tool validation now read the same `SUBAGENT_MODEL` constant as `src/sub_agent.py`, so VLM-swap setups don't silently break inside `<tool>` blocks or new-tool acceptance.
- `examples/score_pair.py --show-chain` hint replaced a dead-end `result['chain']` instruction.
- `examples/inspect_library.py` and `examples/show_reasoning_format.py` verified clean against the current `src/library` API (no signature drift).
- `examples/README.md` &mdash; corrected the "7-entry final library" claim to "6-entry (3 Skills + 3 Tools)", matching what's actually committed at `src/library/`. Same correction landed on the website's Limitations card.
- `TROUBLESHOOTING.md` &mdash; corrected two doc/code drifts: the `/v1/models` health check shows `Qwen2.5-VL-7B-Instruct` (no `Qwen/` prefix), and the OOM mitigation now points at the actual `GPU_MEM` env var (default 0.85) instead of a non-existent `--gpu-memory-utilization` flag with the wrong default.
- `OUTPUTS.md` &mdash; documented the `_about` / `_library_dir` / `_orchestrator` / `_sub_agent` / `average` metadata fields the sample `benchmark_results.json` carries.
- `SECURITY.md` &mdash; "Supported versions" table now reflects v0.1.2 as latest (was still pointing at v0.1.1).
- `examples/show_reasoning_format.py` &mdash; corrected the tool-call limit (`MAX_TOOL_CALLS = 5` in `src/sub_agent.py`; was misdocumented as "default 3, bounded by config").
- `.github/ISSUE_TEMPLATE/bug_report.md` &mdash; the version-collection command `pip show A B C | head -3` was silently dropping the second and third packages; switched to `grep -E '^(Name|Version)'` which surfaces all three.
- `tests/` &mdash; added regression coverage for `REWARDHARNESS_SUBAGENT_MODEL` on all three vLLM call sites (`SubAgent`, `Library.call_tool`, `Evolver._validate_tool_prompt`); a future refactor that re-hardcodes the model id will now fail a test. Suite grew from 100 → 103 tests, still ~2 s.
- Website `script.js` dark-mode-toggle no longer wipes the inline SVG sun/moon icons with a Unicode entity on the first call &mdash; the CSS-based icon swap (already shipped) now works as designed.

## [0.1.2] — 2026-05-16

### Added

- `examples/score_pair.py` &mdash; smallest-possible end-to-end script: Library + Router (Gemini) + SubAgent (vLLM) → 1&ndash;4 preference judgment for a single edit pair.
- `examples/sample_evolution_log.json` and `examples/sample_benchmark_results.json` &mdash; illustrative output files matching the paper's headline numbers; cross-linked from `OUTPUTS.md` so users can `jq`/diff their own runs.
- `.github/dependabot.yml` &mdash; weekly pip + monthly GitHub Actions security tracking.
- `MANIFEST.in` &mdash; ships `score-guidelines/*.md`, `examples/seed_library/`, and `configs/` in the sdist; closes a packaging hole where wheel installs were missing the runtime templates.
- `REWARDHARNESS_TEMPLATES_DIR` env-var escape hatch in `src/sub_agent.py` for unusual install layouts.
- `.editorconfig` for consistent contributor style.

### Changed

- README: new "Swapping in a different VLM as Sub-Agent" section explaining the two pluggability axes (OpenAI-compatible vs subclass); "What you can do with this code" 4-bullet hook near the top; Hardware-requirements table now lists per-workflow credentials.
- Website: new **Reasoning Trace** section showing a real `<think>/<tool>/<obs>/<answer>` chain; **"Why it works"** callout in Method section articulating the context-evolution thesis; brand SVG replaces the cross-platform-flaky 🦞 emoji in the title; Tutorial button replaces the dead `#` self-link.
- `make demo` and `make benchmark` default to `--library-dir examples/seed_library` for non-empty starting state.
- `make help` is now a credentials matrix showing what each target actually needs.

[0.1.2]: https://github.com/TIGER-AI-Lab/RewardHarness/releases/tag/v0.1.2

## [0.1.1] — 2026-05-16

### Security

- **Removed hardcoded internal API key** that was inadvertently shipped in `vanilla/bench_wanqing.py` and three `vanilla/gemini_bench_*.py` scripts in `v0.1.0`. The file `vanilla/bench_wanqing.py` is deleted; the three remaining scripts now read `GEMINI_GATEWAY_BASE_URL` and `GEMINI_GATEWAY_API_KEY` from the environment. See `SECURITY.md` for full disclosure timeline.

### Added

- `SECURITY.md` &mdash; responsible-disclosure policy and supported-version matrix.
- Substantial post-release polish across docs and examples:
  - `WALKTHROUGH.md` (9-step clone-to-first-judgment), `TROUBLESHOOTING.md`, `OUTPUTS.md`, `CONTRIBUTING.md`, `CHANGELOG.md`.
  - `examples/seed_library/` (2 Skills + 1 Tool starter), `examples/show_reasoning_format.py`, `examples/score_pair.py`, `examples/sample_evolution_log.json`, `examples/sample_benchmark_results.json`.
  - `scripts/check_env.py` preflight; `make check` target.
  - `pyproject.toml` for editable install; `.env.example`; `requirements-vllm.txt` split out from the core deps.
  - GitHub issue templates + repo description/topics; CI workflow file prepared but not yet pushed (waiting on workflow scope).

### Changed

- `make demo` and `make benchmark` now default to `--library-dir examples/seed_library` so first-time users get non-trivial output without doing a 4&ndash;6 h evolution first.
- `src/__init__.py` exposes `__version__`.
- README adds a "Swapping in a different VLM as Sub-Agent" guide; Hardware-requirements table now lists credentials needed per workflow.
- Mermaid architecture diagram in README; per-folder READMEs (`tests/`, `examples/`, `vanilla/`, `score-guidelines/`).

[0.1.1]: https://github.com/TIGER-AI-Lab/RewardHarness/releases/tag/v0.1.1

## [0.1.0] — 2026-05-15

Initial open-source release. Paper: [arXiv 2605.08703](https://arxiv.org/abs/2605.08703). Project page: [rewardharness.com](https://rewardharness.com).

### Added

- **Core framework** (`src/`): Orchestrator (Router + ChainAnalyzer + Evolver), frozen Sub-Agent reasoning loop, versioned Skills/Tools Library with snapshot/restore, Phase A/B/C self-evolution pipeline with gated rollback.
- **Reproduction scripts** (`scripts/`): `run_evolution.py`, `run_benchmark.py`, `reproduce.sh` (7-step end-to-end), multi-GPU vLLM launchers (`serve_vllm_multi.sh`, `sbatch_vllm.sh`), `check_env.py` preflight, `setup_env.sh`, `download_data.sh`.
- **Baseline benchmarks** (`vanilla/`): direct VLM scoring on EditReward-Bench / GenAI-Bench / ImagenHub with Claude- and Gemini-backed variants.
- **Test suite** (`tests/`, 100 tests, ~2 s): fully mocked Library / Router / SubAgent / Evolver / Pipeline / Evaluator tests with no GPU / network / API dependencies.
- **Examples** (`examples/`): `inspect_library.py` (Library data-model tour) and `show_reasoning_format.py` (annotated `<think>/<tool>/<obs>/<answer>` trace).
- **Build + packaging**: `Makefile` (install / check / test / demo / evolve / benchmark / reproduce / clean), `pyproject.toml` (editable install), split `requirements.txt` / `requirements-vllm.txt` so CPU-only workflows skip the heavy CUDA dependency, `.env.example`.
- **Docs**: `README.md` with mermaid architecture diagram, Hardware-requirements table, full `default.yaml` reference, CI/coverage-style badges; `WALKTHROUGH.md` (9-step clone-to-first-judgment); `TROUBLESHOOTING.md`; per-folder READMEs for `tests/`, `examples/`, `vanilla/`, `score-guidelines/`; `CITATION.cff` so GitHub renders a "Cite this repository" widget.
- **License**: Apache-2.0.

### Performance (paper headline)

- **47.4%** average accuracy on EditReward-Bench + GenAI-Bench using the Gemini-2.0-Flash Sub-Agent (best K=2: 66.2 / K=3: 45.3 / GenAI: 64.4); **45.7%** with Qwen2.5-VL-7B (best K=3: 46.7 / GenAI: 67.5).
- Surpasses GPT-5 (42.1) by **+5.3** points using only **100 preference demonstrations** (0.05% of the EditReward training data).
- As a reward signal for GRPO fine-tuning of FLUX.2-klein-base-4B, raises ImgEdit-Bench from 3.32 → **3.52**, matching the much larger Flux.1 Kontext [dev].

[0.1.0]: https://github.com/TIGER-AI-Lab/RewardHarness/releases/tag/v0.1.0
