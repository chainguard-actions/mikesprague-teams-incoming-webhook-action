<!-- markdownlint-disable -->

# Hardening Report: mikesprague--teams-incoming-webhook-action/v2.2.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mikesprague--teams-incoming-webhook-action/v2.2.1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Two reusable workflow calls use a mutable branch ref (@main) instead of a pinned 40-character commit SHA, making them vulnerable to supply-chain attacks if the upstream repository is compromised.

- `.github/workflows/build-and-test.yml`: `uses: mikesprague/reusable-workflows/.github/workflows/pages-deploy.yml@main`
- `.github/workflows/dependabot-auto-merge.yml`: `uses: mikesprague/reusable-workflows/.github/workflows/dependabot-auto-merge.yml@main`

Locations:

- `.github/workflows/build-and-test.yml:208`
- `.github/workflows/dependabot-auto-merge.yml:8`

### script-injection (severity: high)

Sub-rule (a): GitHub Actions expressions are interpolated directly inside `run:` shell command strings, allowing template substitution to inject arbitrary shell content before the shell ever parses the command.

In the 'Prepare release data' step of create-release.yml, `${{ github.repository }}` is interpolated directly into the shell script:
```
full_changelog_url="https://github.com/${{ github.repository }}/compare/${last_tag}...${new_tag}"
```

In the 'Generate Release via CLI' step, `${{ steps.prep_release.outputs.is_prerelease }}`, `${{ steps.prep_release.outputs.full_changelog_url }}`, and `${{ steps.prep_release.outputs.new_tag }}` are all interpolated directly into the shell script:
```
if [[ "${{ steps.prep_release.outputs.is_prerelease }}" == "true" ]]; then
  ...
gh release create "${{ steps.prep_release.outputs.new_tag }}" -t "${{ steps.prep_release.outputs.new_tag }}" ...
echo "**Full Changelog:** ${{ steps.prep_release.outputs.full_changelog_url }}"
```
These values should be passed via `env:` variables and referenced as `$ENV_VAR` in the shell script.

Locations:

- `.github/workflows/create-release.yml:93`
- `.github/workflows/create-release.yml:122`

### github-env-injection (severity: high)

The 'Prepare release data' step in create-release.yml writes values to `$GITHUB_OUTPUT` without applying the required sanitization (`printf '%s' ... | tr -d '\n\r'`) before each write.

1. `full_changelog_url` is constructed from `${{ github.repository }}` (a `github.*` context value interpolated directly in the run block) and written to `$GITHUB_OUTPUT` unsanitized:
```
echo "full_changelog_url=$full_changelog_url" >> "$GITHUB_OUTPUT"
```

2. `new_tag` is derived from `${GITHUB_REF_NAME}` (an inherited process environment variable, treated as untrusted per the check scope) and written to `$GITHUB_OUTPUT` unsanitized:
```
echo "new_tag=$new_tag" >> "$GITHUB_OUTPUT"
```

A newline character embedded in any of these values could allow injection of additional key=value pairs into the GitHub output file, potentially overwriting other outputs or environment variables consumed by downstream steps.

Locations:

- `.github/workflows/create-release.yml:100`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings across three workflow files:

1. **unpinned-uses** in `.github/workflows/build-and-test.yml` and `.github/workflows/dependabot-auto-merge.yml`: Both `@main` references to `mikesprague/reusable-workflows` were pinned to commit SHA `73ef64e1657d201843498e58b4847ad4c30fbab8` with a `# main` comment for readability.

2. **script-injection** in `.github/workflows/create-release.yml`: 
   - 'Prepare release data' step: `${{ github.repository }}` moved to `env: GITHUB_REPOSITORY_NAME` and referenced as `$GITHUB_REPOSITORY_NAME` in the shell script.
   - 'Generate Release via CLI' step: `${{ steps.prep_release.outputs.is_prerelease }}`, `${{ steps.prep_release.outputs.full_changelog_url }}`, and `${{ steps.prep_release.outputs.new_tag }}` moved to `env:` block as `IS_PRERELEASE`, `FULL_CHANGELOG_URL`, and `NEW_TAG`, referenced as plain shell variables.

3. **github-env-injection** in `.github/workflows/create-release.yml`: All values written to `$GITHUB_OUTPUT` are now sanitized using `printf '%s' "$VAR" | tr -d '\n\r'` before writing, preventing newline injection attacks.

