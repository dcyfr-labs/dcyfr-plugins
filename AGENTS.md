# AGENTS.md

`dcyfr-plugins` is the official, curated plugin registry for the `@dcyfr/ai` framework, run by DCYFR Labs. **It publishes metadata and trust artifacts only (manifest, SBOM, trust score), not plugin source code.** There is no `package.json`, no build and no test suite; the checks are JSON validation steps in one GitHub Actions workflow.

## Layout

```
marketplace.json                       index: 12 entries, each with plugin_id, name, version, description, author, license,
                                       capabilities, trust_score, dimensions, verified, audit_date, path
plugins/<name>/.claude-plugin/manifest.json   plugin manifest (12 plugins, all MIT)
plugins/<name>/sbom.json               CycloneDX 1.4 bill of materials
plugins/<name>/trust-score.json        overall, dimensions, recommendation, rationale, last_audited, auditor
plugins/<name>/README.md               only code-quality-checker, secret-detector and vulnerability-scanner have one
.github/workflows/plugin-security-scan.yml   the only workflow
README.md, CONTRIBUTING.md, SECURITY.md, LICENSE
```

The 12 plugins: accessibility-checker, api-schema-validator, code-quality-checker, dependency-auditor, design-token-validator, license-compliance, performance-budget, sbom-generator, secret-detector, test-coverage-gate, tlp-classifier, vulnerability-scanner.

## Schemas, as the CI reads them

- **Manifest** (`plugins/<name>/.claude-plugin/manifest.json`) must have non-empty `plugin_id`, `name`, `version`, `description`, `author`, `license`, `capabilities`, `permissions`. Existing manifests also carry `entrypoint` (`src/index.ts`), `minFrameworkVersion` and `keywords`. `permissions` has `read`, `write`, `network` (boolean) and `filesystem`.
- **License** must be one of `MIT`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `0BSD`.
- **`sbom.json`** must exist next to every manifest.
- **`trust-score.json`** needs a numeric `overall`; below 70 fails the scan.
- **`marketplace.json`** entries need `plugin_id`, `trust_score` and a non-empty `capabilities`.

## How a plugin gets in

A maintainer, or an outside contributor by fork and pull request, adds `plugins/<name>/` with the files above and an entry in `marketplace.json`. The scan runs on every pull request, then a DCYFR Labs maintainer reviews and merges (`CONTRIBUTING.md`). Community submissions go to `dcyfr-labs/dcyfr-community-plugins` instead.

## Commands

There are no local scripts. To replay what CI does, run the node snippets from the workflow against your checkout, or use these read-only checks (Node 20 or newer):

| Check | Command |
|---|---|
| Workflow lint | `actionlint .github/workflows/plugin-security-scan.yml` (clean on `main`) |
| `marketplace.json` parses and counts | `node -e "const m=require('./marketplace.json');console.log(m.plugins.length)"` (prints 12) |
| Registry and plugin files agree | for each `marketplace.json` entry, compare `version` and `plugin_id` with its manifest, `trust_score` with `trust-score.json` `overall`, and the SBOM component version. All 12 agree on `main` as of 2026-11-17. |

## CI (checks a PR must pass)

`plugin-security-scan.yml` runs on every pull request (no path filter, on purpose: both jobs are required status checks, and a filtered-out workflow never reports) and on pushes to `main` that touch `plugins/**` or `marketplace.json`. Required checks on `main`: **`Security Scan`** and **`Validate marketplace.json`**.

- `Security Scan`: manifest required fields, license allow-list, SBOM presence, trust score of at least 70, then a PR comment with the result (needs `pull-requests: write`).
- `Validate marketplace.json`: the three per-entry fields listed above.

## Gotchas and known contradictions (do not paper over them)

- **Threshold:** CI enforces a trust score of **70**; `README.md` states a minimum of **85** and a weighted scoring table. The CI does not compute a score, it reads `overall` from the file. `performance-budget` (85) is the lowest in the registry.
- **`CONTRIBUTING.md` lists checks the workflow does not run:** secret detection, vulnerability scan (npm audit and Grype) and permission validation. Only the four steps above exist. Do not claim the others in docs or PR text.
- **`CONTRIBUTING.md` describes `src/`, tests and a `package.json` per plugin;** this registry holds none of them. The README's "Registry Structure" section is the accurate one.
- Keep `marketplace.json`, `manifest.json`, `trust-score.json`, `sbom.json` and the README plugin table aligned when you change a version or score. Nothing in CI checks that they agree.
- `trust-score.json` is described as automated and not to be edited by hand. Change it only as part of an audit.
- `marketplace.json` `last_updated` and `audit_date` are hand-maintained dates.
- Files marked `TLP:CLEAR` are public; keep markers as they are.

## Never

- Add plugin source code, binaries or install scripts to this repository.
- Raise a trust score or `verified` flag without an audit.
- Add a license outside the allow-list, or a plugin that requests `network: true` without a maintainer's explicit approval.
- Weaken the scan or remove a required check to get a PR through.
- Commit secrets or tokens.
- Push to `main`.
