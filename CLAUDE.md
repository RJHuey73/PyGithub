# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What This Is

**PyGithub** (`RJHuey73/PyGithub`, a fork of `PyGithub/PyGithub`) is a Python
library that wraps the [GitHub REST API](https://docs.github.com/en/rest) (and
part of the GraphQL API). It exposes one Python class per GitHub API resource
(`Repository`, `Issue`, `PullRequest`, `NamedUser`, …), each generated/kept in
sync with GitHub's official OpenAPI specification. Entry point:
`github.Github`, instantiated with an `Auth` strategy, exposing `get_*`
methods that return PyGithub objects or `PaginatedList`s.

```python
from github import Github, Auth
g = Github(auth=Auth.Token("access_token"))
for repo in g.get_user().get_repos():
    print(repo.name)
```

**Two in-repo docs already cover architecture and conventions in depth — read
them before making non-trivial changes, don't re-derive from scratch:**

- **`ARCHITECTURE.md`** — the canonical guide to class hierarchy
  (`CompletableGithubObject` vs `NonCompletableGithubObject`), lazy loading,
  the `_initAttributes`/`_useAttributes` attribute pattern, the `Requester`
  HTTP layer, `PaginatedList`, `NotSet` vs `None`, the exception hierarchy,
  file layout/import order, docstring conventions, and a full "Quick
  Reference: Adding a New PyGithub Class" checklist.
- **`OPENAPI.md`** — how to navigate the GitHub OpenAPI spec alongside this
  codebase: schema/path lookup, the class↔schema and method↔path docstring
  conventions, and step-by-step rules for mapping a new OpenAPI path to a
  PyGithub class, method name, return type, and parameters.

## Layout

| Path | Purpose |
|------|---------|
| `github/` | The library. ~150+ files, one PyGithub class per GitHub resource (`Repository.py`, `Issue.py`, `PullRequest.py`, …), plus core infra: `GithubObject.py` (base classes/attribute helpers/`NotSet`), `MainClass.py` (the `Github` entry point), `Requester.py` (HTTP layer), `PaginatedList.py`, `Auth.py`, `GithubException.py`, `GithubRetry.py`, `Consts.py`. |
| `tests/` | One test file per PyGithub class (`tests/Repository.py`, `tests/Issue.py`, …), mirroring `github/`. `tests/Framework.py` is the shared `unittest` base class. `tests/ReplayData/` holds recorded HTTP request/response fixtures (see Testing below). |
| `scripts/` | Dev tooling, notably the OpenAPI-sync workflow (see below) and `fix_headers.py`, `prepare_release.sh`, `prepare-for-update-assertions.py`, `update-assertions.sh`. |
| `openapi/` | Tests and CLI plumbing for `scripts/openapi.py` itself (`openapi/Cli.py`, `openapi/conftest.py`, `openapi/cli-sequence`). |
| `doc/` | Sphinx docs source: `development.rst`, `testing.rst`, `openapi.rst`, `scripts.rst`, `github_objects/` (per-class API reference), `examples/`. Built via `sphinx-build doc build`. |
| `requirements/` | Split dependency lists: `test.txt`, `types.txt` (mypy + stubs), `docs.txt`, `scripts.txt` (deps for `scripts/openapi.py` itself, e.g. `libcst`). |
| `.claude/skills/openapi/`, `.claude/skills/sorted-classes/` | Claude Code skills wrapping the OpenAPI-sync and class-sorting workflows described below — prefer invoking these over reinventing the steps. |

## Commands

```bash
pip install -r requirements/test.txt      # test deps
pip install --editable .                  # install this checkout for manual/exploratory use

pytest tests/                             # full test suite
pytest tests/Repository.py                # one file
pytest tests/Repository.py -k testCompare # one test
pytest -k Repository.testCompare          # equivalent, by class.method

tox -epy313                               # run tests via tox (matches CI's per-version matrix)

pip install -r requirements/types.txt
mypy github tests                         # static typing (see pyproject.toml [tool.mypy])

pip install pre-commit
pre-commit run --all-files                # ruff --fix, black, docformatter, pyupgrade, codespell, EOF/whitespace fixers

pip install -r requirements/docs.txt
sphinx-build doc build                    # build Sphinx docs
```

CI (`.github/workflows/ci.yml`) builds the package (`build`, via `twine
check`), runs `tox` across Python 3.10–3.15 (Ubuntu, plus 3.10 on
Windows/macOS) as the `test` matrix (aggregated into one required status by
`test_success`), and runs a dedicated `pytest openapi/Cli.py` job (`openapi`)
for the OpenAPI script itself. Three more jobs run only `on: pull_request`
and diff parts of the OpenAPI-sync tooling (below) between base and head:
`index` ("Check verbs", `:calls:` docstring HTTP-verb drift), `schemas`
("Add schemas", missing OpenAPI schema references), and `implementations`
("Implement schemas", missing attribute/method implementations) — each
prints the exact `scripts/openapi.py` commands to fix it locally when it
fails. A `sort` job (all branches, via the `./.github/actions/sort-classes`
composite action) enforces `scripts/sort_class.py`'s canonical class
ordering. `.github/workflows/lint.yml` runs `mypy` and `pre-commit` (via the
`.github/actions/mypy` / `.github/actions/pre-commit` composite actions)
plus a docs build, separately. A third workflow,
`.github/workflows/openapi.yml`, runs on a daily cron (plus every push) and
drives `scripts/openapi-update-classes.sh` against the live GitHub OpenAPI
spec, pushing suggested changes to `openapi/autosync` /
`openapi/autosync-new-classes` branches for review — the automated
counterpart to the manual OpenAPI-sync workflow below. `tox.ini`'s
`[gh-actions]` section maps Python versions to `tox` envs — keep it in sync
with the CI matrix if you touch either.

## Architecture & Conventions (see `ARCHITECTURE.md` for full detail)

- **Every PyGithub class implements exactly two methods**: `_initAttributes`
  (sets every attribute to the `NotSet` sentinel) and `_useAttributes`
  (parses whichever keys are present in the API response). Properties call
  `self._completeIfNotSet(self._attr)` before returning, which lazily fires a
  GET to the object's own URL (`_complete()`) the first time an unset
  attribute is accessed on a `CompletableGithubObject`.
- **`CompletableGithubObject`** (has its own endpoint/URL, can lazily
  complete — `Repository`, `Issue`, `Label`, …) vs **`NonCompletableGithubObject`**
  (embedded-only, no endpoint, always fully populated — `Reaction`,
  `CommitStats`, `GitAuthor`, …). Pick the right base class for a new type.
- **`NotSet` vs `None`** are distinct: `NotSet` (`Opt[T]` params, default)
  means "field omitted from the request"; `None` means "explicitly clear this
  field" where the API supports it. Don't conflate them.
- **Every class documents its OpenAPI schema** in the class docstring
  (`The OpenAPI schema can be found at - /components/schemas/...`), and every
  HTTP-issuing method documents its call with a `:calls:` line (`` `PATCH
  /repos/{owner}/{repo}/... <docs-url>`_ ``) — both are checked/consumed by
  tooling (the CI "Check verbs" job, `scripts/openapi.py`), so keep them
  accurate when adding or editing methods.
- **A single `edit()` method**, not writable attributes, is the convention
  for mutating an object (see `doc/Design.md`) — explicit about when an API
  call happens.
- **Two-import pattern**: runtime attribute construction needs `import
  github.X` (to avoid circular imports at module load), while type
  annotations use `from github.X import X` inside `if TYPE_CHECKING:` blocks.
  Follow the exact import order in `ARCHITECTURE.md`'s "File Layout" section
  when adding a new class file.

### The OpenAPI-sync workflow (`scripts/openapi.py`, `scripts/sort_class.py`)

This is the primary way classes/methods here get created or updated to match
GitHub's API, not hand-written from scratch:

1. `scripts/openapi.py` reads a downloaded GitHub OpenAPI spec JSON
   (`fetch`), builds a JSON index cross-referencing spec schemas/paths against
   this codebase (`index`), then `suggest`s, `apply`s (add/update attributes
   or methods on an existing class), or `create`s (new class/method) changes.
   Run `python3 scripts/openapi.py --help` / `... COMMAND --help`; use
   `--dry-run` to preview. **Generated code is a suggestion, not guaranteed
   to compile or be correct** — review it.
2. `scripts/sort_class.py index_filename class_name [class_name ...]` then
   sorts a class's attributes/methods into the canonical order the OpenAPI
   script expects (see `ARCHITECTURE.md`'s "Internal Class Order").
3. `scripts/openapi-update-classes.sh` is the umbrella script that drives
   this across many classes at once, one branch per class (or one combined
   branch), optionally creating new classes (`--create-classes`).
4. Two shell helpers, `scripts/get-openapi-path.sh <path>` and
   `scripts/get-openapi-schema.sh <schema-path>` (both require `jq`, both
   read the spec JSON from stdin), do quick spec lookups without touching
   the index.

Full command reference: `OPENAPI.md`, `doc/scripts.rst`, and the
`.claude/skills/openapi/` / `.claude/skills/sorted-classes/` skills.

## Testing

- `unittest`-based (via `tests/Framework.py`), run through `pytest`, with the
  `responses` library mocking HTTP. Each GitHub API call PyGithub makes in a
  test is intercepted, asserted against expectations, and answered with
  **pre-recorded replay data** rather than hitting the live API — one text
  file per test method under `tests/ReplayData/<ClassName>.<testMethod>.txt`.
- **Adding/changing a test that calls the API requires new replay data**:
  `pytest -k Repository.testCompare --record` records
  `tests/ReplayData/Repository.testCompare.txt` — commit that file alongside
  the test. Reuse another test's replay file via `self.replayData("Other.txt")`
  instead of duplicating fixtures for near-identical calls.
- **Authenticated recording** needs a `GithubCredentials.py` at repo root
  (gitignored) with `oauth_token`/`jwt`/`app_id`/`app_private_key`; a test
  class opts into a non-default auth mode via `self.authMode = "jwt" |
  "app" | "none"` in `setUp`.
- **After changing replay data, assertions often need updating too** — rather
  than hand-editing expected values, use the paired scripts:
  `python ./scripts/prepare-for-update-assertions.py` (flattens a test
  method's multi-line assertions) → `./scripts/update-assertions.sh
  tests/Repository.py testCompare` (rewrites expected values from actual) →
  `pre-commit run --all-files` (re-wraps into multi-line form). See
  `doc/testing.rst` for the full loop.
- `pyproject.toml`'s `[tool.pytest.ini_options]` sets `python_files =
  "tests/*.py"` and ignores `openapi/` by default (that suite runs in its own
  CI job: `pytest openapi/Cli.py`).

## Gotchas

- `pyproject.toml` pins `[tool.mypy] python_version = '3.12'` and sets
  `check_untyped_defs`/`disallow_untyped_defs` for the `github.*` module
  override — new code under `github/` needs full type annotations, checked
  with `mypy github tests` (not just `mypy github`).
- `[tool.ruff] unfixable = ["F401"]` — unused-import removal is deliberately
  *not* auto-fixed by `pre-commit`'s ruff hook; you have to remove unused
  imports by hand (likely to avoid silently breaking the two-import
  TYPE_CHECKING pattern above).
- `mypy` runs *outside* pre-commit in `tox -e lint` (`tox.ini`) specifically
  because pre-commit's isolated venv lacks the project's runtime deps and
  type stubs — don't expect `pre-commit run mypy` to work standalone.
- `[tool.setuptools] packages = ["github"]` — only the top-level `github`
  package ships; `github/py.typed` and `*.pyi` stub files are included via
  `[tool.setuptools.package-data]`, so this is a `py.typed`, type-annotated
  library by design.
- Deprecations should use `typing_extensions`'s `@deprecated` decorator
  rather than silently removing attributes/methods — see `doc/development.rst`.
- The "Check verbs" CI job (`ci.yml`) diffs the OpenAPI index's inferred HTTP
  verbs between a PR's base and head — if you edit a `:calls:` docstring line
  incorrectly (wrong verb, malformed RST link), this job fails even if tests
  pass.
