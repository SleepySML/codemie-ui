# Quality Gates

Run order is fastest-to-slowest.

## Full runs

| Command | Covers | Source |
|---|---|---|
| `npm run check:pre-commit` | generated marketplace types up to date (regenerate + git diff --exit-code src/types/generated/), typecheck, lint | package.json::check |
| `npm run build` | typecheck, then vite build | package.json::build |
| `npm run test:coverage` | unit project with coverage, then integration project without coverage | package.json::test |
| `npm test` | both Vitest projects | package.json::test |
| `MR pipeline` | external pre-ci-guard template; waits on ci-pipeline and compliance-report contexts defined outside this repository; no-op for targets other than main | .gitlab-ci.yml::wait-ci-init |

---

### lint

**Run**: `npm run lint`
**Pass**: exit 0
**Fail**: exit 1 with eslint error lines (exit 2 on a configuration error)
**Auto-fix**: npm run lint:fix

### typecheck

**Run**: `npm run typecheck`
**Pass**: exit 0, no output
**Fail**: non-zero exit with TS error lines

### license-headers

**Run**: `npm run license-headers:check`
**Pass**: exit 0
**Fail**: exit 1 listing files without the header
**Auto-fix**: npm run license-headers:fix

### unit-tests

**Run**: `npm run test:unit:slnt`
**Pass**: exit 0 and a Test Files summary with no failures
**Fail**: non-zero exit with FAIL lines; a suite that fails to import adds nothing to the test counter
**Notes**: A suite that fails to import adds nothing to the counter; quote the Test Files line

### integration-tests

**Run**: `npm run test:integration:slnt`
**Pass**: exit 0
**Fail**: non-zero exit with FAIL lines
**Skip if**: Changes are purely in standalone utility functions with no component or store involvement. When in doubt, run anyway.
**Notes**: Integration tests verify the Component → Store → API flow; fix the implementation or the assertion.

### secrets

**Run**: `npm run secrets:check`
**Side effects**: pulls the gitleaks image on first run
**Pass**: exit 0
**Fail**: exit 1 with 'No container engine found' when no engine exists, exit 1 with '<engine> daemon is not running' when it is stopped, gitleaks' non-zero exit otherwise (printed as 'Secrets detected!')
**Notes**: A credential that reached a commit must be rotated. Read the output, not the exit code: the script prints `Secrets detected!` whenever the child exits non-zero, including when the image could not be pulled, and it scans the working tree only, never git history — security/verification.md.

### license

**Run**: `npm run license-check`
**Side effects**: Writes nothing to the repository: no --out or --files flag, so output goes to stdout (license-checker@25.0.1 bin/license-checker). When license-checker is not installed locally, npx downloads the latest registry version into the npx cache over the network, so the checker version is unpinned. Scans the installed node_modules tree (dev dependencies included), not package-lock.json, so a stale or partial install checks the wrong tree.
**Pass**: Exit 0 with the licence summary on stdout, when every scanned package (root package and dev dependencies included) has a licence string containing at least one allow-list entry as a substring (license-checker@25.0.1 lib/index.js uses indexOf, not an exact or SPDX match; a multi-licence array matches by exact element). Exit 0 alone does not prove the scan ran: a read error is printed to stderr as 'Found error' plus the error (license-checker@25.0.1 bin/license-checker) and the process still prints the summary and exits 0.
**Fail**: Exit 1 at the first package whose licence matches no allow-list entry; stderr shows 'Package "<name@version>" is licensed under "<licence>" which is not permitted by the --onlyAllow flag. Exiting.' (license-checker@25.0.1 lib/index.js). Only that first package is reported and no summary is printed.
**Notes**: Changing the allow list is a decision, not a fix. Behaviour read from license-checker@25.0.1 (unpinned; fetched by npx).

### sonar

**Run**: `npm run sonar-local`
**Side effects**: runs the unit coverage and integration suites, writes coverage/, uploads the analysis for the current branch to the Sonar host
**Pass**: exit 0 after 'Sonar scan completed successfully.'
**Fail**: exit 1 on any thrown error: missing config, unresolved branch, coverage failure, scanner or quality-gate failure
**Skip if**: SONAR_TOKEN unset, or the Sonar host unreachable (exit 0 without scanning; config and branch are still read first)

## Pre-commit Gate (Automatic)

The Husky pre-commit hook runs these automatically on every `git commit`:

| Gate | Command | Notes |
|---|---|---|
| Staged file lint/format | `npx lint-staged` | Runs prettier and eslint --fix on staged code files, then `tsc --noEmit` over the whole project, and adds missing licence headers under `src/` (`package.json::lint-staged`) |
| License headers | `npm run license-headers:check` | Checks all source files for license header |
| Secret detection | `npm run secrets:check` | Scans for accidentally committed secrets |
| Sonar local scan | `npm run sonar-local` | Exits 0 without scanning when `SONAR_TOKEN` is unset or the Sonar host is unreachable; otherwise runs `npm run test:coverage` first, then `sonar-scanner` waiting on the quality gate (`scripts/sonar/run-local-sonar.mjs::main`) |

These run automatically — you do not invoke them manually before committing.

---

## Full Pre-MR Checklist

```bash
npm run lint                   # Lint all files
npm run typecheck              # TypeScript check
npm run test:unit:slnt         # Unit tests
npm run test:integration:slnt  # Integration tests
```

All four must exit 0 before pushing and opening an MR. Then run `npm run test-harness` (`README.md` § Sanity UI suite) and the scenarios in [testing/local-verification.md](testing/local-verification.md).
