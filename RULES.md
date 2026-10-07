# garage49 build and release rules

One set of rules for every repository built on the garage49 build farm (Owner, 2026-10-07).
The shared workflows in this repository (`garage49/.github`) implement them; a repository does not
re-implement them.

## Where a repository lives

A repository belongs in the **garage49** organization when it needs builds on the build farm or is
public. A private repository that needs neither stays under `dirty49374`.

## Packages and images

- **npm scope `@garage49` for everything.** A private package is published only to the internal
  registry (verdaccio), a public package to npmjs. Verdaccio proxies
  `@garage49` to npmjs, so the internal registry serves both. Only garage49 npm org members can
  publish into the scope on npmjs, so outsiders cannot plant a same-named package. A private and a
  public package must never share a name.
- **Docker images** go to the internal registry. For a
  public repository it is decided per repository: the internal registry (built on the farm for
  `main` and tags only) or GHCR.

## Making a repository public

- A repository goes public as a **new repository** with fresh history, not by flipping the
  visibility of the private one. Nothing from the private history becomes public.
- Before the first public push, its content is scanned for secrets (gitleaks) and for internal
  hostnames and addresses; internal operating notes stay out of it.
- Pull-request builds of a public repository run on GitHub-hosted runners (free and unmetered for
  public repositories); only `main` and tag builds may use the build farm.

## Triggers

| Event | Does | Publishes |
|---|---|---|
| push to `main` | test + build every target | workflow artifacts (7 days), snapshot docker image if the repo has one |
| pull request | test + build | nothing |
| tag `vX.Y.Z` | test + build | GitHub Release assets, release docker image, package (npm …) |

- **Only a tag releases.** No branch push or merge ever publishes a release.
- **A release tag must be on `main`.** The workflow checks that the tagged commit is an ancestor of
  `main` and fails otherwise (GitHub Free does not enforce branch protection on private repos).
- Public repositories never run pull-request builds on the self-hosted runners (a fork's pull
  request could run code on them); they use GitHub-hosted runners.

## Versions

- **Release version**: SemVer `X.Y.Z`, from the tag `vX.Y.Z`. The project's version file
  (`Cargo.toml`, `package.json`) must say the same; the release fails if it does not. Bump the file
  in a commit on `main`, then tag that commit. Before 1.0, a breaking change bumps MINOR.
- **Snapshot version** (every non-tag build): `X.Y.(Z+1)-dev.N+g<sha>`, where `vX.Y.Z` is the last
  release tag, `N` the commits since it and `<sha>` the 7-character commit. It sorts after the last
  release and before the next one (the Go pseudo-version idea). Before any release:
  `<declared>-dev.N+g<sha>` with `N` = all commits. CI run numbers are not used: they are per
  workflow and cannot be recomputed from git.
- **Docker tags** cannot contain `+`: a snapshot image is `X.Y.(Z+1)-dev.N-g<sha>`, a release image
  `X.Y.Z`. `latest` is pushed only by a release, and only for services Keel deploys from `latest`.
  Deployments that are not Keel-managed pin a digest.

## Assets

`<name>-<os>-<arch>[.exe]` with `os` in `linux | macos | windows` (never `darwin`; it matches Rust's
`std::env::consts::OS`) and `arch` in `x86_64 | aarch64`, plus `SHA256SUMS` in every release.

## Using it

Each repository has one workflow, `.github/workflows/build.yml`, that calls the shared one:

```yaml
name: build
on:
  push: { branches: [main], tags: ['v*'] }
  pull_request: {}
  workflow_dispatch: {}
permissions: { contents: write }
jobs:
  build:
    uses: garage49/.github/.github/workflows/rust-build.yml@main
    with:
      name: <asset base name>
      package: <cargo package>
      targets: '["linux-x86_64","macos-aarch64","macos-x86_64"]'
```

Versions are computed in one place, `actions/version` (`uses: garage49/.github/actions/version@main`).
This repository is public on purpose (a public repository can only call public reusable workflows),
so nothing internal — hostnames, addresses, credentials — is written here. Internal endpoints and
publish credentials live on the build farm machines (runner environment `INTERNAL_DOCKER_REGISTRY`,
`INTERNAL_NPM_REGISTRY`); GitHub Free does not give private repositories organization variables or
secrets, and the jobs that publish internally run only on the farm anyway.
