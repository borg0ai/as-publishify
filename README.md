# Publishify

Publishify helps agents design and review GitHub Actions CI/CD for repositories that ship CLI tools or desktop apps. It covers three workflow modes: `auto` for full feature-to-development-to-production promotion and release, `half` for feature pull requests plus a manually pushed release tag, and `manual` for feature pull request checks only. It also covers build matrices, signing, GitHub Releases, auto-updaters, and configured package-manager distribution.

## Install

Install the skill for Codex with:

```sh
npx skills add borg0ai/as-publishify --skill publishify --agent codex --copy --yes
```

For local development, run the command from this repository and replace `borg0ai/as-publishify` with `.`.

## Use

Ask Codex to use `$publishify` when creating, refining, or reviewing release workflows. The skill inspects the repository's existing branch, build, signing, and distribution contracts before recommending or changing CI/CD.

The skill package lives in [`publishify/`](publishify/). Its release metadata is in [`publishify/skill-release.json`](publishify/skill-release.json).
