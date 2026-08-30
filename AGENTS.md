# Repository Guidelines

## Project Structure & Module Organization

Core Python package code lives in `openmed/`. Keep existing privacy and registry code in `openmed/core/`; add genuinely new capability in top-level packages already present or planned: `clinical/`, `eval/`, `multimodal/`, `structured/`, `risk/`, and `interop/`. Backend adapters live in `mlx/`, `coreml/`, and `torch/`; the REST surface is in `service/`; Swift code and demos are in `swift/`. Tests are under `tests/unit`, `tests/integration`, and `tests/fixtures`; docs are in `docs/`; examples are in `examples/`; release and maintenance scripts are in `scripts/`.

## Build, Test, and Development Commands

- `uv pip install -e ".[dev]"`: install editable package with test and lint dependencies.
- `uv pip install -e ".[dev,hf]"`: add Hugging Face dependencies for model-related work.
- `.venv/bin/python -m pytest tests/ -q`: run the full test suite before preparing a PR.
- `pytest --cov=openmed --cov-report=term-missing`: run local coverage.
- `make build`: build wheel and sdist via `python3 -m build`.
- `make docs-serve` / `make docs-build`: preview or strictly build MkDocs.
- `cd swift/OpenMedKit && swift test`: run Swift package tests.

## Linting and Formatting

Use the checked-in tooling as the single source of truth. Do not apply ad hoc editor formatting or alternate lint configurations.

Python linting and formatting are owned by Ruff through `pyproject.toml`, `.pre-commit-config.yaml`, and the CI lint job. Before opening a Python-touching PR, run `make format`, `make lint`, and `make format-check`. Use `pre-commit run --all-files` for broad validation, or `pre-commit run --files <paths>` when validating a tightly scoped change.

Swift package formatting is separate from Python. OpenMedKit Swift code uses Apple `swift-format` through `.swift-format`, `scripts/format_swift.sh`, `scripts/lint_swift.sh`, and the Swift package CI workflow. Before opening a PR that touches `Package.swift` or `swift/OpenMedKit`, run `make format-swift` and `make lint-swift`.

Keep `.editorconfig`, Ruff, pre-commit, Swift format scripts, Make targets, documentation, and CI in sync when changing lint policy. A lint policy change should explain the local command, the CI gate, and the expected developer workflow.

## Coding Style & Architecture Rules

Use Python 3.10+, 4-space indentation, and 88-character lines. Prefer `snake_case` for functions and modules, `PascalCase` for classes, and `UPPER_SNAKE_CASE` for constants. Public APIs need Google-style docstrings.

Preserve local-first defaults: no mandatory network calls after model download, no telemetry by default, no raw PHI in logs, caches, temp files, or audit artifacts. Audit reports should use offsets, hashes, provenance, and risk scores rather than plaintext identifiers. Core dependencies must remain permissive-license compatible; GPL, source-available, proprietary, DUA-gated data, UMLS, SNOMED CT, CPT, MIMIC, i2b2, and n2c2 assets must not be bundled. Put restricted integrations behind optional user-supplied keys or out-of-process bridges.

**Standing redaction & error-handling checks (C4 + C9):**
- No error message, exception, log line, or validation diagnostic may include raw values, PHI, manifest field content, or cell/notebook text — use counts, offsets, codes, and hashed identifiers only. Grep staged diff for `text`/`value`/`content` in error paths before review.
- Validation must be value-free (shape/offset/type/enum/canonical label) and must include explicit tests for I/O failures (missing/unreadable/parse error/permission error) asserting the same value-free contract and a stable error code — including the corrupt-`models.jsonl` case in C3. All manifest-derived strings must be trimmed, length-bounded, and canonicalized before use.

## Testing and Release Gates

Pytest discovers `test_*.py` and `*_test.py`. Mark external or end-to-end cases with `@pytest.mark.integration`; mark expensive checks with `@pytest.mark.slow`. Keep bundled tests offline-friendly through mocks or synthetic fixtures.

For privacy, aggregate F1 is not enough. Add tests or fixtures for direct-identifier recall, critical leakage, span integrity, deterministic safety sweeps, date shifting, surrogate consistency, multilingual IDs, and quantized-model recall deltas when touching those paths. Committed golden data must be synthetic; DUA datasets are eval-only and never committed.

### Standing Hardening Gates — derived from maintainer completion reviews

Before marking any task done, verify these invariants mechanically; they supersede any narrower issue description.

1. **Hard-negative sweep (C1):** Before finalizing a hard-negative fixture, run the candidate through *every* production validator/matcher plausibly adjacent to it (not only the named one) and confirm rejection by all.
2. **Span & mask reconstruction (C2):** Reconstruct expected masked output from `gold_spans`; assert hard negatives contribute zero spans for any label and `[start,end)` intervals are disjoint.
3. **Fail-closed envelope (C3 — expanded with #2884):** Verifiers/registries/manifests must explicitly reject: missing required fields, undeclared extra files/keys, unknown enum/cell-type variants (→ canonical `UNKNOWN`), non-string/untrimmed/over-length IDs or descriptions, non-list `target_entities`, **and** must resolve every declared `id` via `models.jsonl` (all member types, not only `model`). On manifest load failure, fail closed — do not swallow the exception. **Negative test required:** mock `models.jsonl` as missing/corrupt/truncated (induce `OSError`/`yaml.YAMLError`/`JSONDecodeError`) and assert the loader raises `EnsembleConfigError`/`EnsembleManifestError` rather than returning an empty set — this is the actual #2884 fallback that was removed. Confirm every other sub-case with its own negative test per case.
4. **Numeric bounds fail closed (C3b — #2884):** Every numeric config (weight, agreement threshold, expansion limit, byte budget) must assert: `not isinstance(x, bool) and isinstance(x, (int,float)) and math.isfinite(x) and lower < x <= upper` (documented range, e.g., `(0.0,1.0]`). Add negative tests for `NaN`, `inf`, `-inf`, `True`/`False`, `"1.0"`, `None`, `0`, `>upper`.
5. **Cross-record consistency (C8 — #2884 new):** For any registry/config with repeated records, prove: member IDs unique, validator names unique, mapping key == `record.family`, and at runtime `set(supplied_sources) == set(configured_members)` (report both `missing` and `unexpected`). Each invariant needs its own failing test.
6. **Manifest privacy filtering (C9 — #2884 new):** Treat every manifest-derived field (id, target_entities, description, validators) as untrusted input even when "real": trim, length-bound (e.g., id ≤200), non-empty, canonicalize labels (`NAME→PERSON`, `ID→ID_NUM`), resolve via manifest, fail if absent, never echo raw values in errors/logs (offsets/codes/hashes only).
7. **Result partition invariant (C5):** Partitioning pipelines must prove: every input appears exactly once, `included ∩ excluded = ∅`, excluded carries explicit `reason`, included carries advisory, public API forwards both.
8. **Action completeness (C5b):** Every declared action (`keep`, `mask`, etc.) needs complete auditable semantics covering the full partition — no implicit passthrough.
9. **Locale & benchmark fidelity (C6):** Fixture locales/Faker providers must match the runtime registry, not linguistic intuition (`te→en_IN` if registered); frozen benchmarks (OM-120) pinned as explicit constant/file and asserted unchanged.
10. **Cross-sibling-PR integration (C10 — narrow-scope CI, #2884):** Enforced by `.github/workflows/registry-verifier-gate.yml` (narrow scope: `openmed/training/**`, `models.jsonl`, `*ensemble*`, `*verifier*`, `*manifest*`). The gate fetches current `master`, rehearses the exact merge-tree (`git fetch origin && git merge-tree $(git merge-base HEAD origin/master) HEAD origin/master` or `git merge origin/master --no-commit`), and verifies co-pending sibling surfaces (`models.jsonl`, `training/__init__.py`, `openmed/training/ensemble.py`, `CHANGELOG.md`) still integrate. If conflict or export-surface change, re-merge `master`, re-validate public API docs 100% and full training suite. Rationale: checklist-only failed for #2884/#2881 — narrow scope contains the cost (docs-only PRs unaffected).
11. **Value-free error handling & changelog — see Coding Style / Commit Guidelines.**

## PyPI Release Publishing Guardrails

The `openmed` PyPI release is tag-driven through `.github/workflows/publish.yml`.
Before changing that workflow, read `docs/release/trusted-publishing.md` and
run `tests/unit/test_publish_workflow_version.py` plus
`tests/unit/release/test_provenance_workflow.py`.

The `PYPI_API_TOKEN` secret is scoped to the GitHub `pypi` environment. Do not
remove the publish job's `environment: pypi` block unless the secret is moved
and the release tests/docs are updated in the same change. Do not switch back
to tokenless Trusted Publishing unless the PyPI `openmed` project is already
configured with a matching trusted publisher for owner `maziyarpanahi`,
repository `openmed`, workflow `publish.yml`, and environment `pypi`.

If `pypa/gh-action-pypi-publish` runs without a non-empty `password`, it falls
back to Trusted Publishing and fails with `invalid-publisher` when the PyPI
publisher is missing. Treat that as a release-blocking configuration regression,
not as a transient PyPI outage. GitHub OIDC/SLSA attestation outages are
separate: the release workflow may make provenance evidence best-effort, but
package build, version checks, `twine check`, artifact upload, and PyPI publish
must still pass before a release is considered done.

## Commit & Pull Request Guidelines

Recent history uses concise imperative commits, often with prefixes such as `fix:`. Keep commits and PRs focused on the requested change. Use the repo-owner Git identity. Do not include assistant/tool/vendor names, co-author trailers, generation footers, or worker attribution in issue text, branch names, commit messages, PR titles, PR bodies, or labels. In particular, never use `codex` or `claude` in branch names, PR titles, PR bodies, issue titles, issue bodies, labels, or other repository metadata. Do not assign issues or PRs unless explicitly asked.

Pull requests should follow `.github/PULL_REQUEST_TEMPLATE.md`: description, change type, tests run, docs or changelog updates when applicable, dependency justification, linked issue, and screenshots or example output for UI/docs changes.

**Standing documentation gate (C7):** Any PR changing user-visible behavior (new verifier rejection, keep/unknown-type handling, redaction rewording, fixture/locale/numeric bounds, registry shape) must include a `CHANGELOG.md` entry under `Unreleased` and, where the public surface changed, a strict type/API check. Missing entry = blocking finding.

## Reviewing Existing PRs After Lint Changes

Do not start review on a stale branch when a repo-wide lint or formatting baseline has landed. First require the branch to update from latest `master` by rebasing or merging `master` into the PR branch.

After the branch is updated, run the canonical formatter on that branch and commit only files that belong to the PR's actual scope. Do not run broad formatting on a stale branch before updating from `master`; that creates avoidable churn and makes review harder.

Expect overlapping PRs to need small formatting follow-up commits only for files they already touch. They should not gain the full baseline diff. If updating a PR produces unrelated formatting-only changes outside its scope, stop and ask before reviewing or committing those changes.

Before approving or merging another contributor's PR, verify that its branch is current enough for CI, the relevant lint commands pass, and the diff does not contain unrelated formatting churn.

## Tracked Files and Commit Safety

Do not add untracked files to git unless the user explicitly asks for those files to be tracked. Never force-add or commit ignored files or ignored directories. Stage only files that are already tracked or files the user has explicitly approved for tracking.

When a file is untracked, ignored, private-looking, generated, or outside the requested scope, leave it alone. If there is any doubt about whether a file should be staged, committed, pushed, or included in a pull request, stop and ask the user before taking the git action.

## OpenMedKit and Multimodal Work

Swift and Python surfaces should evolve together unless a plan explicitly scopes a feature to one platform. OpenMedKit priorities are on-device document intake, OCR, PII redaction, structured extraction, task packs, model catalog/download/cache UX, and later pose, audio, speech, and sensor pipelines. Any borderline or medical-device-style feature must surface disclaimers and must not auto-trigger clinical decisions. Apple Foundation Models paths must stay on-device only; reject any cloud fallback for PHI workflows.
