# NYCE workflows

Reusable GitHub Actions workflows for NYCE packages. They build and test, check the version, and release to npmjs.com or GitHub Packages. No npm token is stored anywhere.

This repo is public and holds no secrets.

## Use a workflow

Call a workflow from a thin caller file in your package repo. Pin it by full commit SHA and put the version in a comment:

```yaml
jobs:
    build-and-test:
        permissions:
            contents: read
            packages: read
        uses: NYCE-Software/workflows/.github/workflows/build-and-test.yaml@<full commit sha> # v1.0.0
```

Dependabot (`github-actions` ecosystem) proposes pin updates as pull requests. It rewrites the comment only if the comment ends with a tag that points at the pinned commit. So the SHA you pin must be the exact commit a `vX.Y.Z` tag points at.

A called workflow can never have more permission than its caller gives it. Each caller job must grant at least the permissions listed below.

## What your package must provide

| Package script            | Used by                                                               |
| ------------------------- | --------------------------------------------------------------------- |
| `build`                   | build-and-test, release-npmjs, publish-github-packages                |
| `typecheck`               | build-and-test                                                        |
| `lint`                    | build-and-test                                                        |
| `format`                  | build-and-test                                                        |
| `test`                    | build-and-test, release-npmjs, publish-github-packages                |
| `check-version-collision` | version-check, release-npmjs                                          |
| `version:dependabot`      | dependabot-version-bump                                               |

The repo needs `packageManager` in `package.json` (for example `pnpm@11.28.10`) and a `pnpm-lock.yaml`. Installs use `pnpm install --frozen-lockfile`.

## Workflows

### build-and-test.yaml

Installs, builds, type checks, lints, checks formatting and tests. Runs the pull request merge commit.

| Input                 | Default | Meaning                                                                            |
| --------------------- | ------- | ---------------------------------------------------------------------------------- |
| `node-version`        | `24`    | Node.js version.                                                                   |
| `playwright-browsers` | empty   | Browsers to install, separated by spaces, for example `chromium`. Empty is none.   |
| `size-script`         | empty   | A package script to run after the tests, for example a size budget. Empty is none. |
| `github-packages`     | `false` | Also install the owner's scoped packages from GitHub Packages.                     |

Caller grants: `contents: read`, `packages: read`.

### version-check.yaml

Runs `npm run check-version-collision`. Fails if the version in `package.json` is already published. No install step.

| Input             | Default | Meaning                                       |
| ----------------- | ------- | --------------------------------------------- |
| `github-packages` | `false` | Look the version up on GitHub Packages.       |

Caller grants: `contents: read`, `packages: read`.

### dependabot-version-bump.yaml

For Dependabot pull requests. Raises the patch version by running `npm run version:dependabot`, commits it and pushes it to the pull request branch. It does nothing if the branch already changes the version. It never pushes a tag.

No inputs. Caller grants: `contents: write`.

A push made with the job token does not start new workflow runs. If `main` requires status checks, the bump commit has none, and the pull request cannot merge until someone re-runs the checks. Drop this workflow if that gets in the way.

### release-npmjs.yaml

Stages a release on npmjs.com. A maintainer then approves it on npmjs.com.

| Input                 | Default | Meaning                                                                              |
| --------------------- | ------- | ------------------------------------------------------------------------------------ |
| `node-version`        | `24`    | Node.js version. Needs npm 11.15.0 or later, so use 24 or later.                     |
| `playwright-browsers` | empty   | Browsers to install before the tests.                                                |
| `environment`         | `npm`   | GitHub environment for the stage job. The npm trusted publisher names the same one.  |

Caller grants: `contents: read`, `id-token: write`.

Run it from a tag push (`v*`). Two jobs:

1. `build` has no OIDC token. It checks that the ref is a tag in the form `v1.2.3`, that the tag matches `version` in `package.json`, that the commit is on the default branch, and that `repository.url` matches the GitHub repo exactly. It also checks the version is not published yet. Then it installs, builds, tests and packs.
2. `stage` has the OIDC token and runs in the environment. It does not check out code or install anything. It runs `npm stage publish` on the tarball from the first job, with `--provenance`.

The split matters. Any step in a job with `id-token: write` can ask for an OIDC token, including a dev dependency that runs during the tests.

Prereleases such as `v1.2.3-beta.1` are not supported.

npm settings for each package:

- The trusted publisher names the **caller's** file name, not this repo's file. For a caller at `.github/workflows/release-npmjs.yaml`, enter `release-npmjs.yaml`.
- `id-token: write` must be granted in the caller job and in this workflow.
- Runs on GitHub-hosted runners only. npm does not support self-hosted runners for trusted publishing.
- Allow staged publishing on the trusted publisher. Leave direct publishing off.

### publish-github-packages.yaml

Publishes a private package to GitHub Packages with the job's `GITHUB_TOKEN`. Same two-job shape as the npm release: the job with write access runs no project code.

| Input                 | Default | Meaning                                |
| --------------------- | ------- | -------------------------------------- |
| `node-version`        | `24`    | Node.js version.                       |
| `playwright-browsers` | empty   | Browsers to install before the tests.  |

Caller grants: `contents: read`, `packages: write`.

The package name must use the owner's scope, for example `@nyce-software/example`. GitHub Packages has no trusted publishing and no provenance.

## Rules every workflow here follows

- `permissions: {}` at the top. Each job asks only for what it needs.
- `actions/checkout` with `persist-credentials: false`.
- No `pull_request_target`. No `${{ }}` expressions from event data inside `run:`. They go through `env:`.
- Third-party actions are pinned by full commit SHA with the version in a comment.
- `pnpm install --frozen-lockfile`. Dependency install scripts stay blocked unless the package lists them.
- No caches in workflows that publish.

## Working on this repo

`ci.yaml` runs [actionlint](https://github.com/rhysd/actionlint) and [zizmor](https://docs.zizmor.sh/) on every pull request and push to `main`. Run both locally with `actionlint` and `zizmor .`.

To release a new version of the workflows, tag the commit `vX.Y.Z` on `main`. Callers pin that commit, and Dependabot reads the tag.

Dependabot waits seven days before it proposes a new version of an action. The actionlint image digest in `ci.yaml` is updated by hand.
