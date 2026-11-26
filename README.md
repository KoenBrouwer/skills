# skills

Agent skills for [OpenCode](https://opencode.ai) and other agents that read `SKILL.md` files.

| Skill | What it does |
|---|---|
| [update-npm-packages](skills/update-npm-packages/SKILL.md) | Updates npm dependencies with npm-check-updates, verifies, and opens a PR |

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```sh
npx skills add KoenBrouwer/skills --skill update-npm-packages
```

Add `-g` to install globally.

Or copy or symlink a skill directory into your agent's skills directory. For OpenCode:

```sh
ln -s "$PWD/skills/update-npm-packages" ~/.config/opencode/skills/update-npm-packages
```

Start a new session so the skill is picked up.

## update-npm-packages

Updates npm dependencies with [npm-check-updates](https://github.com/raineorshine/npm-check-updates) (ncu), verifies the result, and opens a pull request. Every judgment call is put to you; nothing is decided silently.

### Usage

Ask the agent in a repo:

- "Update the npm packages"
- "Run ncu"

Or across several repos:

- "Update npm packages in all repos under ~/code/my-org"

### What it does

1. Asks which ncu target to use: `minor`, `patch` or `latest`. It asks on every run.
2. Creates branch `chore/update-npm-packages-YYYY-MM-DD` off up-to-date `origin/<default>`.
3. Installs, then records the audit counts.
4. Bumps with ncu. It uses the global `ncu` when installed, otherwise `npx npm-check-updates`. It adds `--packageManager pnpm` for pnpm repos and `--workspaces` for monorepos.
5. Asks per pinned package (exact versions, overrides) whether to update, keep pinned or skip, showing the version gap and the git history of the pin.
6. Lists available major versions that were not applied, and asks per package which to attempt.
7. Runs the repo's lint, typecheck, unit tests and build. It does not run e2e.
8. When a check fails, it bisects, reverts the offending packages and re-verifies.
9. Commits the bump.
10. Checks for deprecated and unused or obsolete packages, asks per package, applies the choices, verifies, and commits separately.
11. Pushes and opens a PR that lists:
    - updated packages in a table (package, from, to, patch/minor/major)
    - reverted packages
    - pinned package decisions
    - majors left open
    - obsolete and deprecated package decisions
    - audit counts before and after

### Exclusions

The skill honors an existing `.ncurc.*` file. Without one, it drafts a `.ncurc.yml` rejecting engine-bound packages such as `node` and `@types/node`. It shows the draft first and commits it with the bump only after you approve.

### Pinned packages

A pin may have been set unknowingly, so the skill never assumes it is deliberate. Pins are:

- exact versions in `dependencies` or `devDependencies`
- entries in `overrides`, `resolutions` or `pnpm.overrides`

It asks about every pin on every run, including packages already in a `reject` list. Per package you choose:

- **Update:** then choose an exact version or a `^` range (range recommended).
- **Keep pinned:** then choose whether to persist it in `reject` in `.ncurc.yml`.
- **Skip this run.**

A pin whose update is also a major gets a single question.

### Obsolete and deprecated packages

After the bump is committed, the skill checks:

- **Deprecated:** direct dependencies flagged deprecated on the registry (`npm view <name>@<version> deprecated`).
- **Unused:** `depcheck` hits, verified against config files, scripts and CI. Uncertain ones are marked "verify manually".
- **Built-in replacements:** a short list (`node-fetch`, `rimraf`, `mkdirp`, `uuid`), only when the repo's Node engine supports the replacement.

Per package it recommends removing or swapping, after verifying the replacement exists and is maintained. You choose Remove, Swap, Keep or Skip this run. Keep and Skip are never persisted.

A swap is executed only when mechanical. Otherwise it is left as a follow-up in the PR. The changes go into a separate commit in the same PR.

### All mode

1. Takes the repo paths you give, or scans a parent directory for repos with a `package.json` and a git remote, and asks you to confirm the list.
2. Starts one parallel subagent per repo.
3. Collects their summaries.

Subagents cannot ask questions. They return the `.ncurc.yml` draft, the pinned packages with evidence, the list of majors and the obsolete and deprecated packages to the main agent, which asks you.

### Requirements

- `git` and the GitHub CLI (`gh`), authenticated.
- `npm` or `pnpm`, matching the repo's lockfile.
- An agent with a question tool and subagent support (for all mode).
- Optional: a global `ncu`. Without it, `npx` is used.

## License

[MIT](LICENSE)
