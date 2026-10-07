# garage49/.github

Shared GitHub Actions for every garage49 repository, and the rules they implement.

- `RULES.md` — where repositories live, triggers, versions, assets, packages, going public
- `.github/workflows/rust-build.yml` — reusable build/release workflow for Rust programs
- `actions/version` — the one place a build's version is computed

A repository calls a shared workflow from its own `.github/workflows/build.yml`; see `RULES.md`.
Maintained by the sdfj_kr agent for the Owner. Public on purpose: nothing internal is written here.
