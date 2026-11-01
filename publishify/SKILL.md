---
name: publishify
description: Design, refine, and review GitHub Actions CI/CD for repositories shipping CLI tools or desktop apps. Use when changing or auditing pull request gates, branch promotion, build matrices, signing, GitHub Releases, framework auto-updaters, or package-manager distribution workflows.
license: MIT
metadata:
  version: "0.1.12"
---

# Publishify

Shape repository CI/CD around its real branch, build, signing, and distribution contracts. For repositories shipping both CLI and desktop app, treat them as separate products with separate artifacts and install/update paths. Inspect before editing. Preserve working release behavior and fail closed when required release proof is missing.

## Use this skill for

- Pull request verification from feature branches into development branches.
- Promotion pull requests from development branches into production branches, including version checks.
- Release pipelines triggered by production branch updates or release tags.
- Cross-platform desktop builds, signing, notarization, update metadata, and package-manager pull requests.
- Reviewing existing GitHub Actions for gaps, unsafe permissions, broken dependencies, or release races.

Do not use this skill to perform a one-off application release, bump package versions without CI changes, or publish artifacts manually. Use the repository's release skill or documented release workflow for those tasks.

## Workflow

At the start, ask which workflow mode to use: `auto`, `half`, or `manual`. Apply that mode to the repository's actual branch names and release setup.

| Mode | Pull requests and promotion | Release |
| --- | --- | --- |
| `auto` | Run full CI for feature PRs into the development branch. After merge, automate versioning and the promotion PR from development to production. | Production merge triggers the configured release pipeline. |
| `half` | Run full CI for feature PRs directly into production. | Do not create tags automatically. The user pushes a `v*` tag to trigger the configured release pipeline. |
| `manual` | Run full CI for feature PRs directly into production. | Do not configure release automation. |

`auto` is the full pipeline. `half` automates PR validation and waits for a manually pushed release tag. `manual` stops after feature-to-production PR validation. Do not add a development branch, version PR, promotion PR, tag creation, or release job unless the selected mode requires it.

1. **Map repository contract.** Read applicable `AGENTS.md`, all `.github/workflows/**`, local composite actions, package/build manifests, release configuration, version sync tools, release docs/RFCs, and Git status. Identify protected branches, PR base branches, release trigger, version authority, supported OS/architectures, CLI and GUI entrypoints, artifact names, updater requirements, secrets, and package distribution target. Treat CLI and GUI as separate deliverables; do not assume same target support or package format. Treat dirty user changes as owned; do not overwrite them. Detect each shipped product (CLI or app) and its stack from concrete signals such as `tauri.conf.json`, `wails.json`, `Cargo.toml` binaries, or `package.json` `bin`, then load its profile from `references/profiles/` (`tauri.md`, `wails.md`) and name it. A stack with no profile is unverified: check official docs before writing its jobs.
2. **Describe current flow.** Trace events → jobs → `needs` dependencies → artifacts → release/promotion side effects. Record which jobs are required checks and which jobs have write permissions. Never infer support from a matrix name alone; verify actual build targets and artifact output.
3. **Choose smallest safe change.** Keep existing workflows when they already satisfy the requested behavior. Separate PR validation, promotion/versioning, and production release when their triggers or permissions differ. Prefer reusable workflows only when duplication is real. State any platform or secret gap before making edits.
4. **Implement branch gates for selected mode.** Treat `pull_request.branches` as a base-branch filter; it does not restrict PR source branches. Run full CI for every feature PR targeting the selected mode's production branch, or development branch in `auto`. In `auto`, require the production PR source to be the same-repository development branch; automate versioning and the development-to-production PR only when the repository supports that policy. In `half` and `manual`, feature PRs target production directly and need no development-source guard. Keep required check names stable. Do not run publishing jobs for untrusted pull request code. Use [references/versioning-and-bumping.md](references/versioning-and-bumping.md) for event, token, idempotency, and tool-specific requirements when `auto` needs version automation.
5. **Implement release trigger for selected mode.** In `auto`, trigger release from the configured production update or approved release tag, matching repository policy. In `half`, release only from a manually pushed `v*` tag. In `manual`, do not configure release automation. For enabled releases, build one release workflow for all products (jobs in `references/versioning-and-bumping.md` section 7); create the GitHub Release only after every build and verification succeeded. Pin the release commit SHA; do not build an ambiguous moving branch head. Serialize releases where jobs mutate the same release. Grant write permissions only to jobs that create releases or PRs.
6. **Build supported targets for enabled releases.** In `auto` and `half`, derive separate CLI and GUI matrices from verified product support and native dependency constraints. Build every supported OS/architecture and installable format. Upload immutable, clearly named GitHub Release assets and checksums. A successful compile is not proof of installability; inspect actual installers, archives, executable permissions, runtime dependencies, and updater metadata. Read [references/desktop-cli-release.md](references/desktop-cli-release.md).
7. **Sign, notarize, and prepare updater for enabled releases.** Treat OS code signing/notarization and updater artifact/manifest signing as different checks. Inspect the project's framework version, current official framework docs, configured update provider, client endpoint, and release assets. Before enabling macOS release, verify the release GitHub Environment exists, the trusted release job references it, and every required Apple secret is configured at that environment or another intentionally scoped level. Derive secret names and required values from the actual workflow/framework auth method; never assume one universal naming scheme. Check names with GitHub settings or `gh secret list` (which shows names only), and add a workflow preflight that maps secrets to environment variables and fails on missing/empty values without printing them. Validate the Developer ID Application certificate and private key can be imported on the macOS runner, the identity is valid and matches the expected Team ID, and the selected `notarytool` authentication can authenticate before building/publishing release assets. Keep credentials in GitHub secrets or protected environments; expose them only to trusted release jobs. Do not expose them to pull-request workflows, forks, or untrusted code. For macOS direct distribution, verify Developer ID signing and notarization/stapling where required. Fail release when required signatures, notarization, update metadata, or platform artifacts are missing. Test an installed previous version updating from published release before claiming updater support. Read [references/desktop-cli-release.md](references/desktop-cli-release.md) for the Apple credential checklist.
8. **Open package-manager PRs for enabled releases.** Follow the repository's named tap/feed and target branch; do not assume a provider, formula format, cask format, or branch. Inspect the destination repository's conventions and verify the base branch exists. If it does not, stop and report the missing branch; never silently fall back to a default branch or push to the base. Make package-manager updates the last release job: download public assets, verify checksums, update the package definition, and open a release-specific PR with a token scoped to that destination. Package-manager upgrades are separate from in-app auto-updates. See [references/desktop-cli-release.md](references/desktop-cli-release.md).
9. **Use `gh` for GitHub state.** Check `gh auth status` and `gh repo view --json nameWithOwner,defaultBranchRef` before GitHub operations. Read current rules with `gh api repos/{owner}/{repo}/rulesets --paginate` and effective branch rules with `gh api repos/{owner}/{repo}/rules/branches/{branch}`. Read workflow runs and PR checks with `gh run list`, `gh run view`, and `gh pr checks`. Never print tokens or secret values; `gh secret list` exposes names only.
10. **Configure branch rules with `gh`.** Use `gh auth status` and `gh api` for every branch-rule read and write. Do not use the GitHub web UI. Read repository rulesets with `gh api repos/{owner}/{repo}/rulesets --paginate` and effective branch rules with `gh api repos/{owner}/{repo}/rules/branches/{branch}`. Preserve existing rules and bypass actors. Prepare and review a complete JSON payload before writing. Create with `gh api repos/{owner}/{repo}/rulesets -X POST --input <ruleset.json>`; update a discovered ruleset by ID with `gh api repos/{owner}/{repo}/rulesets/{ruleset_id} -X PUT --input <ruleset.json>`. Ruleset writes require repository Administration write permission. If API access fails, report the exact permission/API error and stop; do not fall back to the web UI. Re-read rulesets and effective branch rules after mutation. See [GitHub rulesets REST API](https://docs.github.com/en/rest/repos/rules) and [`gh api` manual](https://cli.github.com/manual/gh_api).
11. **Validate workflow changes.** Run available YAML/action linters and repository validation commands. Check expression syntax, event filters, `permissions`, secret exposure, job dependencies, matrix coverage, artifact names, and failure conditions. Use local workflow simulation only if the repository already supports it. Do not claim hosted workflow success from static validation.
12. **Review final diff and remote state.** Confirm only requested workflow/config/docs paths changed, no secrets entered files or logs, release jobs cannot run from untrusted PRs, and failed build/sign/notary/artifact checks block publication. Report local validation and remote GitHub state separately.

## Diagnose post-job cache failures

When a GitHub Actions run fails during `Post job cleanup` with `Path Validation Error: Path(s) specified in the action for caching do(es) not exist`, inspect the exact failing job log with `gh run view <run-id> --job <job-id> --log-failed`. Identify the action named by the failed post step; do not infer the owner from nearby warnings. A preceding runtime deprecation warning such as Node's `DEP0040` is not the cause unless the log marks it as an error or the job exits there.

For `actions/setup-node@v5`, automatic package-manager caching is enabled when a valid `packageManager` field is detected in `package.json`, even when the workflow omits an explicit `cache` input. If the job only validates metadata and does not need dependency caching, set `package-manager-cache: false` on that setup-node step. If caching is needed, verify the selected manager, cache path, lockfile, and setup/install steps create the required path before post-job save. Avoid installing dependencies solely to satisfy an unused cache in a lightweight guard. Preserve the explicit cache behavior of build/test jobs that benefit from it. See the [setup-node v5 caching documentation](https://github.com/actions/setup-node/blob/v5.0.0/README.md#caching-global-packages-data).

Validate the fix with a new hosted run for the affected PR/job. Static inspection can confirm the workflow input, but cannot prove GitHub's post-job cache save succeeded.

## Mode flow model

```text
auto:
  feature --PR + full CI--> develop --automated version/promotion PR--> main
  main update --> release pipeline

half:
  feature --PR + full CI--> main
  manually pushed v* tag --> release pipeline

manual:
  feature --PR + full CI--> main
```

Implement only the arrows for the selected mode and verify every configured trigger. Follow repository branch names and release policy when they differ from `develop` and `main`.

## GitHub branch protection setup

Workflow YAML cannot enable repository branch protection. Configure it through GitHub CLI and the Rulesets API. Do not use the web UI.

1. Run `gh auth status` and `gh repo view --json nameWithOwner,defaultBranchRef`. Stop if the authenticated account or repository is wrong.
2. Read all rulesets with `gh api repos/{owner}/{repo}/rulesets --paginate`. Read effective rules for each target branch with `gh api repos/{owner}/{repo}/rules/branches/{branch}`. Preserve existing rules, bypass actors, and unrelated branch patterns.
3. Select target branches from the chosen mode. `auto` protects production and development branches. `half` and `manual` protect production only. Require pull requests, block force pushes, and block deletion on each protected branch. Require the CI checks that run for PRs targeting that branch. For `auto`, require the production source-branch guard too; it must pass only when the PR head is the same-repository development branch. Do not add that guard to `half` or `manual`.
4. Get required-check names from successful GitHub workflow runs and PR checks. Use exact stable context names. Never guess names, copy names from unrelated repositories, or require a check that does not run for that target branch. Enable up-to-date branches or merge queue requirements only when their workflows support them and reruns are acceptable. Keep release write permissions on trusted release jobs; PR jobs stay read-only and receive no signing secrets.
5. Prepare the complete Ruleset API JSON with the actual branch patterns, required checks, enforcement settings, and existing bypass actors. Before writing, show that payload's effective summary and get authorization for any rule that can block merges or direct pushes. Create with `gh api repos/{owner}/{repo}/rulesets -X POST --input <ruleset.json>`. Update the discovered ruleset by ID with `gh api repos/{owner}/{repo}/rulesets/{ruleset_id} -X PUT --input <ruleset.json>`. Writes require repository Administration permission.
6. Re-read rulesets and effective branch rules with `gh api`. Confirm target branches, required PR/check rules, source guard when `auto`, bypass actors, and force-push/deletion settings match the reviewed payload. If API access or permission fails, report the exact error and stop. Do not fall back to the web UI.

GitHub's base-branch filter does not enforce source-branch policy. In `auto`, the required source-branch workflow check enforces development-to-production PRs; the Ruleset makes that check mandatory. Report branch protection as unconfigured until the API readback confirms it. Do not claim a workflow file alone enforces branch protection.

When user asks to configure branch rules, use this order:

Follow the six steps above. Verify `auto` source-branch enforcement with PR checks: production PR passes only for the same-repository development branch; other branches and fork branches with the same name fail. Read the [repository Rulesets API](https://docs.github.com/en/rest/repos/rules) and [`gh api` manual](https://cli.github.com/manual/gh_api) for endpoint and payload details.

For pull request creation, inspect destination repository, head branch, base branch, asset URLs, and checksums first. In `auto`, workflow jobs may create the promotion PR. For tap PRs, use a release job. Outside a workflow, use `gh pr create` only when the user requested it. Do not merge unless explicitly asked.

## Safety rules

- Inspect current worktree and preserve unrelated edits. Never reset, clean, or stage broad paths.
- Never weaken required checks to make a workflow green. Report platform jobs deliberately disabled and why.
- Never invent a release version or bump every feature PR by default. Follow the repository's version authority and bump timing; if none exists, propose a concrete policy before wiring automation.
- Never expose signing credentials to fork PRs or untrusted workflow contexts.
- Never publish unsigned artifacts when signing/notarization is part of the product contract.
- Never update a Homebrew formula/cask before release asset URL and checksum are verifiable.
- Never push directly to a package tap's protected/default/target branch; the release job submits a PR to the configured target (default `develop`).
- Treat release creation, asset upload, and tap PR as separate stages. Verify each stage independently and report partial completion.
- Do not invent action inputs, branch names, secrets, artifact paths, supported platforms, or successful checks. Verify against repository config and official action documentation when an action's behavior is uncertain.
