---
name: update-npm-packages
description: Update npm dependencies with npm-check-updates, verify the result, and open a PR. Use when asked to update packages, bump dependencies, run ncu, or refresh dependencies in one repo or across several repos.
---

# Update npm packages

Bump dependencies with `npm-check-updates` (ncu), verify, commit, push and open a PR.

## Modes

- Default: the current repo only.
- `all`: several repos. The user gives repo paths, or a parent directory to scan.

## Rules that always apply

- Never write credentials into files. If private packages need an `.npmrc`, copy the repo's example file when there is one (for example `.npmrc.example`) and leave credentials to the environment.
- If a registry or internal host is unreachable, stop and tell the user what failed. Do not retry through other routes.
- Do not touch `node_modules`, build output, or generated files.
- Read the repo's own agent instructions and README (`AGENTS.md`, `CLAUDE.md`, `README.md`) first. They win over this skill on conflicts.

## Step 0: ask the target (every run)

Ask the user with the question tool which ncu target to use:

- `minor` (minor + patch)
- `patch`
- `latest` (includes majors)

Do not assume a default.

## All mode: select repos

1. Collect the repo paths the user gave. If the user gave a parent directory, list its subdirectories that have a `package.json` and a git remote.
2. Show the list and ask the user to confirm or edit it. Exclude repos that mainly hold committed build output unless the user insists.
3. Ask the target once (Step 0) and pass it to every subagent.
4. Start one parallel subagent per confirmed repo. Each prompt must contain: the repo path, the chosen target, the full procedure below and the rules above.
5. Collect each subagent's summary (PR link, bumped, reverted, majors left, audit delta, failures) and report them together.

Subagents cannot ask the user questions. For anything that needs a decision (the `.ncurc.yml` draft, the pinned-package questions, the per-package major choice, the obsolete and deprecated package choices), the subagent stops before that step and returns the draft or list with evidence. The main agent asks the user, batched per repo, and sends the answers back to the same subagent.

## Procedure per repo

### 1. Prepare

1. Check `git status`. If the tree is dirty, stop and report.
2. Detect the package manager from the lockfile: `pnpm-lock.yaml` → pnpm, `package-lock.json` → npm.
3. If the repo needs an `.npmrc` for private packages and has none, copy its example file if there is one.
4. Fetch and branch off the default branch: `git fetch origin`, then `git switch -c chore/update-npm-packages-YYYY-MM-DD origin/<default>` using today's date.
5. Install: `npm ci` or `pnpm install --frozen-lockfile`.
6. Record the audit counts: `npm audit` or `pnpm audit`. Failures from the audit command are data, not errors.
7. Detect a monorepo (`workspaces` in `package.json`, or `pnpm-workspace.yaml`). If so, pass `--workspaces` (plus `--root` when the root has dependencies) to ncu.

### 2. Exclusions: `.ncurc.yml`

ncu reads `.ncurc.yml`, `.ncurc.json` or `.ncurc.js`. Always write new config files as YAML: `.ncurc.yml`.

If a config exists, honor it and do not rewrite it.

If none exists, draft `.ncurc.yml` and show it to the user before writing:

1. Read `package.json`: `dependencies`, `devDependencies`, `peerDependencies`, `overrides`, `resolutions`, `pnpm.overrides`, `engines`, `packageManager`.
2. Put only engine-bound packages in `reject`: `node`, `@types/node`, and anything tied to `engines`.
3. Do not put pinned or overridden packages in the file. They are handled by the pinned-package question in step 3.
4. Show the draft in chat with the reason for each entry.
5. Write it only after the user approves. It is committed in the same commit as the bump.

Example shape:

```yaml
reject:
  - "@types/node"
```

### 3. Pinned packages

Never assume a pin is deliberate. It may have been pinned unknowingly. Always ask the user.

A package is pinned when it is:

- listed with an exact version (no `^` or `~`) in `dependencies` or `devDependencies`, or
- listed in `overrides`, `resolutions` or `pnpm.overrides`.

Engine-bound packages (`node`, `@types/node`) are not asked about; they stay rejected.

Ask about every pinned package, including those already in a `reject` list in `.ncurc.*`.

1. Collect the pinned packages.
2. For each, gather evidence:
   - Current version → latest version, and the latest version within the chosen target.
   - Git history of the pin: `git log -S"<name>" -- package.json` or `git blame` on the line. Show commit, date and message.
3. If the latest version is a major bump, merge this question with the major question from step 5. Ask once, with the pin evidence and the major decision together.
4. Ask the user per package, with these choices:
   - Update
   - Keep pinned
   - Skip this run
5. For "Update", ask whether to keep the version exact or convert to a `^` range. Recommend the range.
6. For "Keep pinned", ask whether to persist it in `reject` in `.ncurc.yml`. Decide per package, every run. Write nothing without an answer. If not persisted, exclude it for this run only via `--reject`.
7. "Skip this run" behaves like "Keep pinned" for this run, without the persistence question.
8. Apply the updates:
   - Dependencies: `ncu --target <target> -u --filter <package>` (add `--target latest` for an approved major). For a `^` range, edit the version in `package.json` to the new version with a `^` prefix.
   - Overrides, `resolutions` and `pnpm.overrides`: ncu does not touch these. Edit the entry in `package.json` to the chosen version, then install.
9. Install (`npm install` or `pnpm install`) after the edits.
10. Treat each updated pin as its own bisect candidate in step 6.

### 4. Bump

1. Use the global `ncu` if `command -v ncu` finds it. Otherwise use `npx npm-check-updates`.
2. Add `--packageManager pnpm` for pnpm repos.
3. Reject every pinned package for this step, so ncu does not bump pins the user has not approved: `--reject <name1>,<name2>`. Packages the user chose to update in step 3 are already handled.
4. Preview first: `ncu --target <target> --reject <pins>` (no `-u`). Show the proposed changes.
5. Apply: `ncu --target <target> --reject <pins> -u`.
6. Install again (`npm install` or `pnpm install`) to refresh the lockfile.

### 5. Majors

Skipped when the target is `latest`.

1. Run `ncu --target latest --reject <pins>` (no `-u`) to list available majors that were not applied. Pinned packages with a major available were already asked in step 3.
2. Show the list and ask the user per package which to attempt.
3. Apply each chosen major with `ncu --target latest -u --filter <package>`, then install.
4. Treat each major as its own bisect candidate in step 6.

### 6. Verify

Use the repo's own scripts from `package.json` and its `AGENTS.md`. Typical order:

1. Lint
2. Typecheck
3. Unit tests
4. Build

Do not run e2e (Cypress, Playwright, Backstop).

If a step fails:

1. Identify the offending packages: read the error first, then bisect by reverting groups of bumps (`git checkout -- package.json <lockfile>`, re-apply a subset, reinstall, re-run).
2. Revert the offenders to their previous version and add them to a "reverted" list with the failing step and short error.
3. Re-run verification until it passes.
4. If nothing can be made to pass, stop and report. Do not push a failing branch.

### 7. Commit the bump

- Commit `package.json` changes, the lockfile, and the approved `.ncurc.yml` if new.
- Follow the repo's commit message convention (check recent `git log`). Default: `chore: update npm dependencies`.
- Do not mention the agent or tool in the message.

### 8. Obsolete and deprecated packages

Runs after the bump is verified and committed. Changes go into a separate commit in the same PR, so they can be reverted alone. Skip the commit when nothing changes.

1. Deprecated: for each direct dependency, run `npm view <name>@<installed version> deprecated`. A non-empty result is the deprecation message. Do not check transitive packages.
2. Unused: run `npx depcheck`. Verify every hit before asking. Check config files, `package.json` scripts, CI files, and tooling conventions (plugins, `@types/*`). Ask normally about confirmed-unused packages. Mark uncertain ones "verify manually" in the question.
3. Built-in replacements: suggest a swap only when the repo's `engines.node` supports the replacement.

   | Package | Replacement |
   |---|---|
   | `node-fetch` | global `fetch` |
   | `rimraf` | `fs.rm` with `recursive` and `force` |
   | `mkdirp` | `fs.mkdir` with `recursive` |
   | `uuid` | `crypto.randomUUID` |

4. Pick the swap target from the deprecation message or your own knowledge. Verify with `npm view` that it exists, was published recently, and is not itself deprecated. If no replacement is found, recommend Remove only when the package is unused. Otherwise recommend Keep.
5. Ask the user per package with the question tool. Show the reason (deprecation message, unused, or built-in replacement) and a recommended action (remove, or swap to X). Choices:
   - Remove
   - Swap
   - Keep
   - Skip this run

   "Keep" and "Skip this run" apply to this run only. Persist nothing.
6. Execute:
   - Remove: `npm uninstall <name>` or `pnpm remove <name>`.
   - Swap when mechanical: install the replacement, then edit imports and usages in the source.
   - Swap when the API differs: stop and record it as a follow-up. Do not guess the migration.
7. Run the verification from step 6. If it fails, bisect and revert the offending removals or swaps like a bump, and list them as reverted.
8. Commit following the repo's convention, for example `chore: remove obsolete and deprecated packages`.

### 9. Audit

Run the audit again. Record the before and after counts for the PR.

### 10. Push and PR

1. `git push --set-upstream origin <branch>` (use `--force-with-lease` on a re-run).
2. Write the PR body. Include:
   - Target used.
   - Table of every updated package, always included, with columns Package, From, To, Type (`patch`, `minor` or `major`). Include pinned updates, applied majors, and swap replacements. Derive Type from the semver difference between From and To. Mark `0.x` changes by the same rule (the changed segment), and add a `(0.x)` note when a minor on a `0.x` package may be breaking.
   - Reverted packages and why.
   - Majors applied and majors left open.
   - Pinned packages: updated, kept pinned, skipped.
   - Obsolete and deprecated packages: removed, swapped, kept, skipped, and non-mechanical swaps left as follow-up, each with its reason.
   - Verification run (lint, typecheck, unit, build) and that e2e was not run.
   - Audit before → after.
   - Whether `.ncurc.yml` was added.
3. Open the PR with `gh pr create` against the default branch.
4. Report the PR URL.

## Report

Lead with the result. Per repo: PR link, bumped count, reverted packages, pinned decisions, majors left open, obsolete and deprecated decisions, audit delta, anything unverified. Do not describe a PR as released or deployed.
