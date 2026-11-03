---
name: dependency-health
description: Recurring dependency chore — scan the default branch for known vulnerabilities (CVE/GHSA), review open Dependabot PRs against their real diffs, cross-reference the two, classify merge safety, and (only when instructed) merge safe PRs or open fix PRs for uncovered vulnerabilities
argument-hint: "[merge safe | merge <pr>... | fix | fix <package>...]"
---

# Dependency Health

Two inputs, one verdict per item:

- **Vulnerabilities** affecting the default branch — from GitHub Dependabot alerts *and* a local audit of the committed lockfiles.
- **Open Dependabot PRs** — each upgrade inspected against its actual version diff and this codebase's usage.

They are cross-referenced: a PR that fixes an advisory is flagged **SECURITY**; an advisory with no PR fixing it is **UNCOVERED**.

Nothing is merged, pushed or changed unless the arguments (or a follow-up reply) ask for it.

> This file is synced from `~/Code/cockpit/skills/dependency-health/SKILL.md` — edit it there, not here.
> Repo-specific overrides go in `NOTES.md` next to this file (never overwritten by sync).

## Usage

```
/dependency-health                    # report only (default)
/dependency-health merge safe         # report, then merge every SAFE PR
/dependency-health merge 3441 3440    # report, then merge these PRs
/dependency-health fix                # report, then open fix PRs for UNCOVERED vulns that have a patched version
/dependency-health fix <package>...   # same, limited to these packages
```

## Ground rules

- **Never touch the user's working tree.** The checkout may be on a feature branch with uncommitted work. No `checkout`, `stash`, `reset`, `pull`, `composer install`, `npm install` in the repo dir. Read default-branch files with `git show origin/$BASE:<path>`; search code with `git grep … origin/$BASE`; write changes only in a separate worktree (Step 7).
- All commands must work from any cwd (the cockpit runs this skill against a repo path). Use `git -C "$REPO_DIR"` and `gh -R "$SLUG"`.
- Never `rm` a path built from a variable. Temp dirs come from `mktemp -d` and are left for the OS to clean.
- When uncertain, downgrade a verdict (SAFE → REVIEW → RISKY).

## Step 0 — Context

```bash
REPO_DIR=<path given by the caller, else: git rev-parse --show-toplevel>
SLUG=$(git -C "$REPO_DIR" remote get-url origin | sed -E 's#.*github.com[:/]##; s#\.git$##')
BASE=$(gh repo view "$SLUG" --json defaultBranchRef -q .defaultBranchRef.name)
git -C "$REPO_DIR" fetch origin "$BASE" --quiet
```

Then:

1. If `NOTES.md` exists next to this file, read it — it overrides anything below.
2. From `origin/$BASE`, read `composer.json` / `package.json` (prod vs dev deps; `composer.json` `"type": "library"` means the lockfile only affects this repo's dev/CI, but **constraints** in `require` matter to consumers) and `.github/dependabot.yml` (watched ecosystems + directories).
3. List committed lockfiles:
   ```bash
   git -C "$REPO_DIR" ls-tree -r --name-only "origin/$BASE" \
     | grep -E '(^|/)(composer\.lock|package-lock\.json|yarn\.lock|pnpm-lock\.yaml)$' \
     | grep -vE '(^|/)(vendor|node_modules)/'
   ```

## Step 1 — Vulnerabilities

### 1a. GitHub Dependabot alerts

```bash
gh api "repos/$SLUG/dependabot/alerts?state=open&per_page=100" --paginate --jq '.[] | {
  alert: .number, pkg: .dependency.package.name, eco: .dependency.package.ecosystem,
  manifest: .dependency.manifest_path, scope: .dependency.scope,
  ghsa: .security_advisory.ghsa_id, cve: .security_advisory.cve_id,
  severity: .security_advisory.severity, summary: .security_advisory.summary,
  range: .security_vulnerability.vulnerable_version_range,
  patched: .security_vulnerability.first_patched_version.identifier }'
```

HTTP 403 "Dependabot alerts are disabled" → record a **config gap** and continue; the local audit becomes the only signal.

### 1b. Local audit of default-branch lockfiles

For each directory containing a `composer.lock` or `package-lock.json`:

```bash
T=$(mktemp -d)
for f in composer.json composer.lock package.json package-lock.json; do
  git -C "$REPO_DIR" cat-file -e "origin/$BASE:$DIR$f" 2>/dev/null \
    && git -C "$REPO_DIR" show "origin/$BASE:$DIR$f" > "$T/$f"
done
( cd "$T" && composer audit --locked --no-plugins --format=json )       # if composer.lock
( cd "$T" && npm audit --package-lock-only --json )                     # if package-lock.json
```

(`$DIR` is the lockfile's directory with trailing `/`, empty for the root.)

- `composer audit` also lists **abandoned** packages — report them as informational.
- `yarn.lock` / `pnpm-lock.yaml` can't be audited without installing; rely on 1a for those and say so in the report.

### 1c. Merge and enrich

Dedupe 1a + 1b by (ecosystem, package, GHSA/CVE). For each vulnerability record:

- installed version on `$BASE` (from the lockfile), vulnerable range, first patched version (or "no fix available")
- **direct or transitive** — composer: `composer why --locked <pkg>` in `$T`; npm: `npm ls --package-lock-only <pkg>` in `$T`
- **prod or dev** — `require` vs `require-dev`, `dependencies` vs `devDependencies` (for transitive: what pulls it in)
- severity (critical / high / moderate / low)

### 1d. Config gaps

Report any of: no `.github/dependabot.yml`; a committed lockfile whose ecosystem/directory isn't covered by `dependabot.yml`; Dependabot alerts disabled. These explain why vulnerabilities would go unnoticed — always list them.

## Step 2 — Open Dependabot PRs

```bash
gh pr list -R "$SLUG" --author app/dependabot --state open --limit 100 \
  --json number,title,url,headRefName,body,labels,mergeable,mergeStateStatus,statusCheckRollup,createdAt
```

For each PR extract:

- **Packages + versions** from the title — formats vary: `chore(deps): bump <pkg> from <a> to <b>`, `Bump <pkg> from <a> to <b>`, `chore(deps-dev): …`, or grouped `Bump the <group> group … with N updates` (packages are listed in the body).
- **Ecosystem** from the branch: `dependabot/composer/…`, `dependabot/npm_and_yarn/…`, `dependabot/github_actions/…`.
- **Semver jump**: patch / minor / major (`0.x` minor bumps count as major).
- **CI** from `statusCheckRollup` — any failing check disqualifies SAFE. Pending checks → note it, don't count as green.
- **Mergeability** — `CONFLICTING` / `BEHIND` → note it (Step 6 handles rebasing).

### Per-PR diff analysis

If you can spawn subagents, use one per PR, in parallel, to keep the main context clean. Otherwise do it sequentially. If there are more than ~10 PRs, analyze in this order and be brief on the tail: SECURITY-relevant (Step 3), major bumps, prod minor, prod patch, dev-only, GitHub Actions.

For each PR:

1. Read the PR body — release notes, changelog excerpts, commit list.
2. Get the real diff between the two versions from the source repo:
   - Find the repo from the PR body links, or `https://repo.packagist.org/p2/<vendor>/<pkg>.json` (`source.url`), or `npm view <pkg> repository.url`.
   - `gh api repos/{owner}/{repo}/compare/{oldTag}...{newTag}` — try `v`-prefixed and bare tags. Read `files[].patch` for small diffs; for large ones focus on `src/` / `lib/` / the package's entry points.
   - GitHub Actions: compare the action repo's tags the same way.
3. Find how this codebase uses it — on the default branch, never the working tree:
   ```bash
   git -C "$REPO_DIR" grep -n -E '<namespace|facade|import|config key|changed symbol>' "origin/$BASE" -- . \
     ':!vendor' ':!node_modules' ':!*.lock' ':!package-lock.json' ':!public/build'
   ```
   Dev-only deps (`require-dev`, `devDependencies`, `deps-dev` in the title) can't affect production behavior.
4. Return: package, old → new, jump, summary of actual changes, which changed symbols this codebase touches, verdict + one-line justification.

## Step 3 — Cross-reference

For each vulnerability from Step 1:

- **Covered** — an open Dependabot PR bumps the package to a version ≥ the first patched version. Tag that PR **SECURITY** (with severity + advisory id).
- **UNCOVERED** — no such PR. State why, one of:
  - no patched version exists yet
  - transitive, and the parent's constraint pins it below the fix (name the parent)
  - our own constraint (`composer.json` / `package.json`) excludes the fixed version
  - ecosystem/directory not watched by `dependabot.yml`, or alerts disabled
  - Dependabot PR was closed/ignored (check `gh pr list -R "$SLUG" --author app/dependabot --state closed --search <pkg>`)
  - unknown — say so

And state the concrete fix: e.g. `composer update <pkg> --with-dependencies`, `npm update <pkg>`, `npm audit fix` (never `--force`), bump parent X to ≥ Y, or widen our constraint to `^Z` (a manual decision).

## Step 4 — Classify PRs

- **SAFE** — CI green, and the diff only touches code paths this codebase doesn't use, or is dev-only tooling, or is a bugfix whose corrected behavior we already expect. Patch bumps of GitHub Actions with no workflow-relevant changes qualify.
- **REVIEW** — changes behavior in code paths we use (even if likely compatible), changes config defaults, or deprecates something we use. Name the affected files/usages and what to verify.
- **RISKY** — major bump, documented breaking changes touching our usage, required config/migration steps, or failing CI. Never merged by this skill without explicit confirmation; list the follow-up work.

**SECURITY** is a tag, not a verdict: a SECURITY PR keeps its SAFE/REVIEW/RISKY verdict, but goes first in the report, and a high/critical SECURITY PR at REVIEW should be recommended for merge after the named verification.

Base verdicts on the actual diff, not the changelog prose.

## Step 5 — Report

In this order:

1. **Header** — `$SLUG` · base `$BASE` · date.
2. **Vulnerabilities** — table sorted by severity, UNCOVERED first within each severity:
   `severity | package | installed → fixed | prod/dev | direct/transitive | advisory | coverage (PR #N / UNCOVERED: reason)`
3. **Dependabot PRs** — table, SECURITY first, then SAFE → REVIEW → RISKY:
   `PR # | package(s) | old → new | jump | prod/dev | CI | verdict | tags`
   followed by a short justification for every non-SAFE PR.
4. **Config gaps** and **abandoned packages** (if any).
5. **Summary block** — always emit exactly this fenced block last (the cockpit parses it):

   ````
   ```json dependency-health-summary
   {"repo": "<SLUG>", "base": "<BASE>",
    "vulns": {"critical": 0, "high": 0, "moderate": 0, "low": 0, "uncovered": 0, "uncovered_high_or_critical": 0},
    "prs": {"total": 0, "safe": [], "review": [], "risky": [], "security": []},
    "config_gaps": [], "alerts_enabled": true}
   ```
   ````
   (`safe`/`review`/`risky`/`security` are lists of PR numbers.)

If no action was requested, end by asking which PRs to merge (recommend "all SAFE + SECURITY") and whether to open fix PRs for UNCOVERED vulnerabilities that have a patched version. When run by the cockpit, skip the question — the cockpit asks once for all repos.

## Step 6 — Merge (only when instructed)

Merge one PR at a time — parallel merges conflict on lockfiles. Use squash if the repo allows it:

```bash
gh repo view "$SLUG" --json squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed
gh pr merge <n> -R "$SLUG" --squash   # or --merge / --rebase per repo settings + history
```

- Merge only SAFE PRs, or PRs the user named regardless of verdict — for a named RISKY PR, confirm once first.
- After a merge, Dependabot rebases the others. If a later merge fails because the branch is behind/conflicting, comment `@dependabot rebase` on it (`gh pr comment <n> -R "$SLUG" --body "@dependabot rebase"`) and report it as **deferred** — don't retry in the same run.
- Report merged / skipped / deferred.

## Step 7 — Fix uncovered vulnerabilities (only when instructed)

Only for UNCOVERED vulnerabilities that have a patched version and where the fix stays inside existing constraints. Constraint widening or parent upgrades are listed, not done, unless the user named that package.

```bash
STAMP=$(date +%Y%m%d)
WT="$(mktemp -d)/wt"
git -C "$REPO_DIR" worktree add -b "chore/security-$STAMP" "$WT" "origin/$BASE"
cd "$WT"
composer update <pkg>... --with-dependencies --no-install --no-scripts   # composer: lockfile only
npm update <pkg>... --package-lock-only                                  # npm direct
npm audit fix --package-lock-only                                        # npm transitive (never --force)
```

1. Re-run the Step 1b audit inside `$WT` — confirm the advisories are gone and nothing new appeared. Check `git diff --stat` — the lockfile should be the only change (plus `composer.json`/`package.json` only if the user approved a constraint change).
2. Commit (`chore(deps): security update <pkgs> (<GHSA/CVE ids>)`), push, open the PR against `$BASE` with a body listing each advisory, old → new version, and anything left unfixed. Add a `dependencies` / `security` label only if it already exists in the repo.
3. `git -C "$REPO_DIR" worktree remove "$WT"`.
4. Report the PR URL. Never merge a fix PR in the same run — CI has to run on it first.

## Notes

- Don't run the test suite locally — verdicts rely on CI plus diff analysis. Local verification of a REVIEW PR is a separate follow-up.
- Repos with `composer.json` `"type": "library"`: a vulnerable lockfile entry is CI/dev exposure only; also check whether our `require` constraints *allow* consumers to resolve to a fixed version — that is the real consumer-facing issue.
