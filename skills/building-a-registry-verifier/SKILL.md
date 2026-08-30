---
name: building-a-registry-verifier
description: "Build a fail-closed registry/verifier for manifests, member configs, or bundle verifiers — manifest-resolved members, bounded numerics, duplicate/family/runtime invariants, privacy-filtered fields, and merge-tree rehearsal. Use when adding a YAML/JSON registry, ensemble config, model manifest verifier, or bundle validator. Trigger on 'registry', 'verifier', 'ensemble', 'manifest validation', 'teacher-ensemble', or 'bundle verifier'."
license: Apache-2.0
metadata:
  project: OpenMed
  category: evaluation-quality
  pairs: adjacent
  version: "1.0"
---

# Building a Registry Verifier — fail-closed from the first commit

Verifier and registry work (#2893 bundle/file integrity, #2884 config/registry semantics) share a category — fail-closed structural validation — but differ in mechanism. This skill exists on its own merits, for the same reason `building-gold-corpus` exists: the category now has enough discrete invariants to warrant a worked checklist and a reference example rather than a single AGENTS.md bullet.

## When to use

- You are adding or changing a YAML/JSON registry, ensemble config, or bundle/manifest verifier.
- You touch `models.jsonl`, `teacher_ensemble.yaml`, bundle manifests, or any `*verifier*`/`*registry*` loader.

## Reference implementation

`openmed/training/ensemble.py@ab6a6c9` (maintainer-hardened #2884) — bounded numerics, trim/length/canonicalize, manifest resolution for all member types, duplicate/family/runtime checks, corrupt-manifest fail-closed, and merge-tree rehearsal after #2881.

## Worked checklist — copy into your PR description and tick

- [ ] Every `id`/`family`/`type`/`description` validated: `str`, trimmed, non-empty, `len(id) ≤200`
- [ ] Every numeric: `not bool`, `isinstance((int,float))`, `math.isfinite`, `0 < x ≤ 1.0` (or doc range); negative tests for `NaN`/`inf`/`True`/`"1.0"`/`None`
- [ ] Every member resolved via `models.jsonl` (all types, not only `model`); canonicalize labels; load failure → exception, not empty-set fallback
- [ ] **Corrupt-manifest negative test:** mock `models.jsonl` as missing/corrupt/truncated (`OSError`/`yaml.YAMLError`/`JSONDecodeError`) and assert loader raises `EnsembleConfigError`/`EnsembleManifestError` (not empty set) — the actual #2884 fallback
- [ ] Cross-record: duplicate member IDs rejected, duplicate validators rejected, `key == record.family`, runtime `supplied == configured` (report `missing`/`unexpected`)
- [ ] Fail-closed scalars: missing required fields, undeclared files/keys, unknown validators/cell-types → rejection with value-free diagnostic (offsets/codes/hashes only)
- [ ] Privacy-filtered even when "real": no raw manifest/cell text in errors/logs
- [ ] Merge-tree rehearsal (C10): `git fetch origin && git merge-tree $(git merge-base HEAD origin/master) HEAD origin/master` clean; sibling PRs on same export surface re-validated (narrow CI enforces this — see `.github/workflows/registry-verifier-gate.yml`)

## Hand-off

- **From** `building-gold-corpus` — same completeness bar, different domain (fixtures vs. verifiers).
- **To** `gating-deid-leakage` — a verifier gate can be wired as a CI check identical to a leakage gate.
- **Pairs with** the narrow-scope CI gate: `registry-verifier-gate.yml` fires only on `openmed/training/**`, `models.jsonl`, `*ensemble*`, `*verifier*`, `*manifest*` so docs-only PRs pay no cost.

## Standards & references

- OpenMed ensemble loader: `openmed/training/ensemble.py` (`load_teacher_ensemble_config`, `validate_ensemble_against_manifest`)
- Model manifest: `models.jsonl` (canonical source of truth — every declared member must resolve there)
- Merge-tree rehearsal: `git merge-tree $(git merge-base HEAD origin/master) HEAD origin/master`
- Prior hardening: #2893 (bundle verifier — missing/undeclared rejection), #2884 (registry — numeric bounds, cross-record, corrupt-manifest, merge-tree)
