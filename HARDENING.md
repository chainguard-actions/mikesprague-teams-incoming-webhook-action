<!-- markdownlint-disable -->

# Hardening Report: mikesprague--teams-incoming-webhook-action/v2.2.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **mikesprague--teams-incoming-webhook-action/v2.2.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two workflow files reference reusable workflows using a mutable branch ref (@main) instead of a pinned 40-character commit SHA. This means the action could silently change to a malicious version without any review. Affected references: (1) `mikesprague/reusable-workflows/.github/workflows/pages-deploy.yml@main` in build-and-test.yml; (2) `mikesprague/reusable-workflows/.github/workflows/dependabot-auto-merge.yml@main` in dependabot-auto-merge.yml.

Locations:

- `.github/workflows/build-and-test.yml:148`
- `.github/workflows/dependabot-auto-merge.yml:8`

### script-injection (severity: high)

The `create-release.yml` workflow interpolates GitHub Actions expressions directly inside `run:` shell command strings (sub-rule a). The expressions `${{ github.repository }}`, `${{ steps.prep_release.outputs.is_prerelease }}`, `${{ steps.prep_release.outputs.full_changelog_url }}`, and `${{ steps.prep_release.outputs.new_tag }}` are substituted into the shell script before the shell parses it, allowing an attacker who controls these values to inject arbitrary shell commands. These values should be passed via `env:` variables and then referenced as quoted shell variables.

Locations:

- `.github/workflows/create-release.yml:76`
- `.github/workflows/create-release.yml:78`
- `.github/workflows/create-release.yml:100`
- `.github/workflows/create-release.yml:103`
- `.github/workflows/create-release.yml:104`

### github-env-injection (severity: high)

The `Prepare release data` step in `create-release.yml` writes values derived from `$GITHUB_REF_NAME` (an inherited process env var set by GitHub Actions from the git tag ref) and `${{ github.repository }}` to `$GITHUB_OUTPUT` without first sanitizing them with `printf '%s' ... | tr -d '\n\r'`. A git tag name or repository name containing embedded newlines could inject additional key=value pairs into GITHUB_OUTPUT, potentially overwriting outputs consumed by later steps. Affected writes: `echo "new_tag=$new_tag" >> "$GITHUB_OUTPUT"`, `echo "last_tag=$last_tag" >> "$GITHUB_OUTPUT"`, `echo "full_changelog_url=$full_changelog_url" >> "$GITHUB_OUTPUT"`, `echo "is_prerelease=$is_prerelease" >> "$GITHUB_OUTPUT"`.

Locations:

- `.github/workflows/create-release.yml:82`
- `.github/workflows/create-release.yml:83`
- `.github/workflows/create-release.yml:84`
- `.github/workflows/create-release.yml:85`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings:

1. **unpinned-uses** (build-and-test.yml line 148, dependabot-auto-merge.yml line 8): Pinned both `mikesprague/reusable-workflows` references from `@main` to `@6771c975237a2165e414447932e6487d6d4f245e # main`.

2. **script-injection** (create-release.yml lines 76, 78, 100, 103, 104): Moved all `${{ }}` expressions out of `run:` shell strings into `env:` blocks. In `Prepare release data`, `${{ github.repository }}` is now `GH_REPOSITORY` env var. In `Generate Release via CLI`, `${{ steps.prep_release.outputs.is_prerelease }}`, `${{ steps.prep_release.outputs.full_changelog_url }}`, and `${{ steps.prep_release.outputs.new_tag }}` are now `IS_PRERELEASE`, `FULL_CHANGELOG_URL`, and `NEW_TAG` env vars respectively, referenced as plain shell variables.

3. **github-env-injection** (create-release.yml lines 82-85): All four `echo ... >> "$GITHUB_OUTPUT"` writes now sanitize values first using `printf '%s' "$VAR" | tr -d '\n\r'` before writing to prevent newline injection attacks.

