---
name: tuist-github-cache
description: Speed up Tuist build, test and deploy jobs on GitHub Actions. Caches SPM checkouts (`Tuist/.build`) and the SwiftPM download cache (`~/.cache/swifterpm`) with actions/cache keyed on `Tuist/Package.resolved`, wires up Tuist's binary cache (`tuist auth login` via OIDC + `tuist setup cache` + `enableCaching`), adds path filters and concurrency so runs don't waste minutes, and upgrades actions to Node 24 versions. Use when CI `tuist install` / `tuist test` / `tuist build` is slow, when asked to "cache tuist on github actions", "speed up tuist CI", when a cache never hits or stays stale after a dependency bump, or when adding caching to build-test.yml / deploy-ios.yml. Triggers on "tuist github cache", "tuist ci speed", "tuist 깃헙 캐시", "CI 빌드 속도".
---

# Tuist CI speed on GitHub Actions

Three independent layers. Apply in order; each is a separate, small PR.

| Layer | What it saves | Where |
|-------|---------------|-------|
| 1. Dependency cache | `tuist install` re-resolving / re-cloning SPM packages | `actions/cache` |
| 2. Tuist binary cache | recompiling unchanged modules/targets | `tuist setup cache` + `enableCaching` |
| 3. Don't run at all | whole jobs for non-code changes, superseded runs | `paths:` filters, `concurrency:` |

## 0. Inspect first

- `ls .github/workflows/` — find every workflow that runs `tuist install` / `generate` / `build` / `test` (typically build-test and deploy).
- `Tuist.swift` — is `fullHandle` set and `generationOptions: .options(enableCaching: true)` present?
- `Tuist/Package.resolved` — must be **committed**; it is the cache key. If it is gitignored, `hashFiles` returns an empty string and the key never changes.
- Note the runner (`runs-on`) and whether tuist comes from mise (`jdx/mise-action`).
- `ls ~/.cache/swifterpm` locally to confirm that directory is what Tuist's SwiftPM resolver uses on this tuist version.

## 1. Dependency cache (`Tuist/.build` + `~/.cache/swifterpm`)

Put the step **after** tool install (mise) and **before** `tuist install`:

```yaml
- name: Cache Tuist dependencies
  uses: actions/cache@v6
  with:
    path: |
      Tuist/.build
      ~/.cache/swifterpm
    key: tuist-build-v1-${{ runner.os }}-${{ runner.arch }}-${{ hashFiles('Tuist/Package.resolved') }}
```

Rules:

- **Same step, same key in every workflow** (build-test, deploy) so they share one entry.
- **Key on `Package.resolved`** — a dependency bump changes the hash, so a fresh cache is created. Do not add `restore-keys` fallbacks unless you accept a stale partial restore.
- **Changing the `path:` list does not change the key.** An old entry saved with fewer paths is restored as an exact hit and is never re-saved, so the new path stays cold forever. **Bump the key prefix** (`v1` → `v2`) whenever you add or remove a cached path.
- Do not cache `Derived/` or generated `.xcodeproj`; they are cheap to regenerate and go stale.
- Caches are saved only when the job succeeds. A red first run saves nothing.
- Scope: a PR can read caches from its base branch (main) but main cannot read a PR's. Merge the caching PR, then the first push to main seeds the cache for everyone.

## 2. Tuist binary cache

```yaml
permissions:
  id-token: write      # Tuist OIDC auth

steps:
  - run: tuist auth login
  - run: tuist install
  - run: tuist setup cache
  - run: tuist test    # or tuist build / tuist generate
```

`Tuist.swift`:

```swift
generationOptions: .options(enableCaching: true)
```

- `id-token: write` must be on the job (or workflow) or `tuist auth login` fails on CI.
- Keep `tuist auth login` before any command that talks to the Tuist server.
- Binary cache only helps when targets are split into real modules/packages; a single fat app target re-compiles every time.

## 3. Skip and cancel wasted runs

```yaml
on:
  push:
    branches: [ "main" ]
    paths: [ '**/*.swift', 'mise.toml', 'Tuist/Package.resolved', '.github/workflows/build-test.yml' ]
  pull_request:
    branches: [ "main" ]
    paths: [ '**/*.swift', 'mise.toml', 'Tuist/Package.resolved', '.github/workflows/build-test.yml' ]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

- Include the workflow file itself and `mise.toml` in `paths`, otherwise changing CI never gets tested by CI.
- Add `Project.swift`, `Tuist/Package.swift`, `Tuist/Package.resolved`, `*.xcconfig` if they are not already matched by a glob.
- Use `cancel-in-progress` for build/test. For deploy workflows scope the group (e.g. `${{ github.ref }}/ios/deploy`) so a new tag build supersedes only the same ref.
- A path-filtered workflow that is a **required check** will block docs-only PRs forever ("expected"). Make the check non-required or add a no-op fallback job.

## 4. Keep actions current

Old majors run on deprecated Node runtimes and warn. Known-good set at time of writing: `actions/checkout@v7`, `actions/cache@v6`, `jdx/mise-action@v5`. Verify the latest tags before pinning.

## 5. Verify

1. Push the PR. First run: log shows `Cache not found for input keys: tuist-build-v…` and, at job end, `Cache saved with key: …`.
2. Merge to main (seeds the default-branch cache), or re-run the job on the PR.
3. Second run: `Cache restored from key: …`; compare `tuist install` step duration before/after.
4. Dependency bump PR: key changes → miss → new save. That is expected.
5. `gh run list --workflow=build-test.yml` / `gh cache list` to see sizes. Entries over ~10 GB total per repo get evicted LRU; 7 days unused also evicts.

## Pitfalls

- Hit but still slow → the cached path is not the one the tool uses; check the step log for where `tuist install` actually clones.
- Cache never saves → job failed before the post-step, or `path` expands to nothing.
- Stale after changing paths → forgot to bump the key prefix (see §1).
- Deploy workflow not exercised by PR CI → say so in the PR test plan; it only runs on dispatch.
