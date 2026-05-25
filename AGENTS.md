# AGENTS.md

Hermeto is a Python CLI tool that pre-fetches project dependencies for hermetic
(network-isolated) container builds and generates SBOMs (Software Bill of Materials).
It supports 9+ package managers: gomod, pip, npm, yarn, yarn_classic, bundler,
cargo, rpm, and generic.

## Core principles

1. **Report prefetched dependencies accurately** — PURLs must match actual
   artifacts. Platform strings, version qualifiers, and checksums must be exact.
2. **No arbitrary code execution** — never shell out to untrusted tools.
3. **Always validate checksums** — a mismatch fails the entire request.
4. **Favor reproducibility** — use lockfiles, not dependency resolution.

## Architecture

```
hermeto/
  interface/cli.py            # Typer CLI: fetch-deps, generate-env, list-backends
  core/
    resolver.py               # Dispatches to PM handlers via dict lookup
    models/
      input.py                # Request, PackageInput (discriminated union on type)
      output.py               # RequestOutput, BuildConfig, EnvironmentVariable
      sbom.py                 # Component, ExternalReference, Annotation
      property_semantics.py   # Property names and PropertySet
    errors.py                 # Error hierarchy with exit codes and friendly messages
    rooted_path.py            # Path traversal prevention
    checksum.py               # Checksum validation
    scm.py                    # Git operations
    package_managers/
      gomod/main.py           # fetch_gomod_source(request) -> RequestOutput
      bundler/main.py         # fetch_bundler_source(request) -> RequestOutput
      npm/main.py             # fetch_npm_source(request) -> RequestOutput
      pip/main.py             # fetch_pip_source(request) -> RequestOutput
      cargo/main.py           # fetch_cargo_source(request) -> RequestOutput
      yarn/main.py            # fetch_yarn_source(request) -> RequestOutput
      yarn_classic/main.py    # fetch_yarn_classic_source(request) -> RequestOutput
      rpm/main.py             # fetch_rpm_source(request) -> RequestOutput
      generic/main.py         # fetch_generic_source(request) -> RequestOutput
```

### Package manager contract

Every PM implements: `fetch_<pm>_source(request: Request) -> RequestOutput`

The resolver (`core/resolver.py`) maps PM type strings to handler functions:
```python
_package_managers: dict[PackageManagerType, Handler] = {
    "gomod": gomod.fetch_gomod_source,
    "bundler": bundler.fetch_bundler_source,
    ...
}
```

RequestOutput is summed across all PMs using `__add__()`.

### Component and PURL creation

Each PM:
1. Parses lockfile/manifest
2. Extracts name, version, source info
3. Generates PURL via `PackageURL(...).to_string()`
4. Creates `Component(name=..., version=..., purl=..., properties=...)`

PURL accuracy is critical — downstream security scanners and SBOM consumers
rely on PURLs matching actual artifacts.

### Environment variable generation

PMs return env vars as `EnvironmentVariable` objects with `${output_dir}`
template placeholders. These end up in the hermetic build container. Example
from gomod (`core/package_managers/gomod/main.py` line 536):

```python
env_vars_template = {
    "GOCACHE": "${output_dir}/deps/gomod",
    "GOPATH": "${output_dir}/deps/gomod",
    "GOMODCACHE": "${output_dir}/deps/gomod/pkg/mod",
    "GOPROXY": "file://${GOMODCACHE}/cache/download",
}
```

GOSUMDB is set at line 734. Any new env var must also be documented.

### Error handling

Errors use a hierarchy rooted at `BaseError` (`core/errors.py`):
- `UsageError` → `InvalidInput`, `PackageRejected`, `ChecksumVerificationFailed`
- `FetchError`, `PackageManagerError`, `GitError`

Each error has an exit code, a reason, and an optional solution string.
Error messages should be friendly and actionable.

## Package manager gotchas

### bundler
- `GemPlatformSpecificDependency` handles platform-aware gems
- Platform string (e.g., `aarch64-linux-gnu`) must be preserved exactly in
  PURLs and download URLs — Ruby normalizes platforms but PURLs need originals
- See `bundler/gem_models.py` for PURL generation

### gomod
- `GOSUMDB` controls sum database lookups — should be `off` during hermetic
  builds to prevent network access
- Go workspace support (`go.work`) affects module resolution
- Vendor directory handling differs between Go 1.18+ workspaces and single
  modules
- PURL qualifiers include `type=module` vs `type=package`

### npm / yarn
- Workspace hoisting affects dependency resolution
- Dev dependencies are tagged with `npm_development` property

### pip
- Binary wheel filtering via platform/architecture matching
- sdist vs wheel preference logic in download selection

## Build and test

```shell
nox                          # all checks (lint + unit tests)
nox -s lint                  # ruff check, ruff format, mypy
nox -s python-3.10           # unit tests on Python 3.10
nox -s python-3.10 -- tests/unit/package_managers/gomod/  # PM-specific tests
nox -s python-3.10 -- tests/unit/test_cli.py --stepwise   # debug failures
nox -s integration-tests     # build image + run integration tests
nox -s generate-test-data    # regenerate test fixtures
```

## Code style

- Ruff for formatting, import sorting, and linting (line length 100)
- mypy with `disallow_untyped_defs=True` — full type annotations required
- Each new `.py` file needs: `# SPDX-License-Identifier: GPL-3.0-only`
- All tool config is in `pyproject.toml`

## Tests

- Unit tests: `tests/unit/package_managers/<pm>/`
- Integration tests: `tests/integration/`
- Add new test cases, don't modify existing ones
- Prefer `pytest.mark.parametrize` over duplicate test functions
- Never string-match against output
- Cover positive and negative scenarios
- Test data regeneration: `hack/mock-unittest-data/<pm>.sh`

## Commits and PRs

- Write clear commit messages explaining the "why"
- Sign off all commits with DCO: `git commit -s`
- Split changes into self-contained commits
- No gitmojis
- Amend existing commits during review, don't add fixup commits
- Every commit must pass CI independently
- See AI_CONTRIBUTION_POLICY.md for AI disclosure requirements
  (use `Assisted-by:` or `Co-authored-by:` trailers)

## Experimental features

New package managers use the `x-` prefix (e.g., `x-maven`). They need:
- A design document (`docs/design/package-manager-template.md`)
- Small incremental PRs
- An ADR to graduate to production
