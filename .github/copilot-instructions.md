# openedx-translations — AI Agent Instructions

## Project Overview

This is the **central translation repository for all Open edX projects**, implementing [OEP-58](https://open-edx-proposals.readthedocs.io/en/latest/architectural-decisions/oep-0058-arch-translations-management.html). It stores `.po` (Django/gettext) and `.json` (React ICU MessageFormat) translation files for 50+ repos. Downstream projects pull translations via the `openedx-atlas` CLI.

Translations are synchronized with Transifex via the Transifex GitHub App, which auto-uploads source strings and auto-downloads reviewed translations as PRs. The repository has a `main` branch (nightly) and release branches named `open-release/<release-name>.master` (e.g. `open-release/redwood.master`).

## Repo Structure

```
translations/          # All translation files, one subdirectory per upstream repo
  <repo-name>/
    <app-name>/
      conf/locale/<lang>/LC_MESSAGES/   # Python/Django .po files
      src/i18n/messages/<lang>.json     # React ICU .json files
scripts/               # Python automation scripts
  tests/               # pytest tests for scripts; mock_translations_dir/ has fixture files
requirements/          # pip-tools managed; .in = direct deps, .txt = compiled/pinned
.github/workflows/     # GitHub Actions: CI, Transifex sync, validation, release management
transifex.yml          # Transifex GitHub App config (file patterns per project)
package.json           # Node.js dep: @formatjs/cli for JSON validation
Makefile               # All common dev tasks
```

## Build & Test Commands

```bash
# Install Python test dependencies
make test_requirements

# Run all Python tests (pytest, with coverage)
make test
# Equivalent: pytest -v -s --cov=. --cov-report=term-missing --cov-report=html scripts/tests

# Install translation script runtime dependencies
make translations_scripts_requirements

# Validate translation files (.po and .json)
make validate_translation_files

# Install Node.js dependencies (for JSON file validation)
npm clean-install
```

### Running individual scripts

```bash
# Fix Transifex resource names (requires TRANSIFEX_API_TOKEN env var)
make fix_transifex_resource_names RELEASE=main
make fix_dry_run_transifex_resource_names RELEASE=redwood   # dry run

# Sync release project translations on Transifex
python scripts/release_project_sync.py --release-name=<name>

# Retry stalled Transifex bot PRs
bash scripts/retry_merge_transifex_bot_pull_requests.sh
```

## Translation File Conventions

### Python/Django repos — `.po` files

- Path: `translations/<repo>/<app>/conf/locale/<lang>/LC_MESSAGES/<domain>.po`
- Source (English) files live under `/en/LC_MESSAGES/` — **never edit these directly** (extracted by upstream repos).
- Translated files are auto-managed by Transifex; only edit via Transifex UI.
- Validate with GNU `msgfmt`: `msgfmt --check <file>.po`

### JavaScript/React repos — `.json` files

- Path: `translations/<repo>/src/i18n/messages/<lang>.json`
- Format: ICU MessageFormat; validated with `@formatjs/cli verify --structural-equality`.
- `transifex_input.json` files are **source files** — skip validation on them.
- Source locale files (`/en/` or `en.json`) are never directly edited.

## Python Scripts — Key Patterns

- **Auth**: All scripts read `TRANSIFEX_API_TOKEN` from env; fall back to `~/.transifexrc`.
- **`fix_transifex_resource_names.py`**: Renames Transifex resource slugs to human-readable names. Adds a 4-digit random suffix to avoid collisions. Supports `--dry-run` and `--force-suffix`.
- **`validate_translation_files.py`**: Validates `.po` via `msgfmt` subprocess and `.json` via `formatjs` subprocess. Exits `0` on all valid, `1` if any invalid. Supports `--types po`, `--types json`, explicit file/dir args.
- **`release_project_sync.py`**: Syncs resources from the `openedx-translations` main Transifex project to a named release project.

## Testing Conventions

- Framework: **pytest** (not unittest).
- `scripts/` is a Python package (`__init__.py` present); tests use relative imports like `from ..fix_transifex_resource_names import ...`.
- HTTP calls to Transifex API are mocked with the `responses` library — do not make real API calls in tests.
- Use `monkeypatch` for non-deterministic calls (e.g. mock `random.choice` to ensure stable slugs).
- Test fixtures live in `scripts/tests/mock_translations_dir/` — add valid and intentionally invalid files there to test all validation paths.
- Coverage: HTML report written to `htmlcov/`.

## Branching Model

- `main` — latest/nightly translations, corresponds to Transifex project `openedx-translations`.
- `open-release/<name>.master` — translations pinned to a named release. Corresponding Transifex project: `openedx-translations-<name>`.
- Use `RELEASE=<name>` in Makefile targets to target a specific release branch/project.

## CI/CD — What Runs and When

| Workflow                               | Trigger                                             | Notes                                                       |
| -------------------------------------- | --------------------------------------------------- | ----------------------------------------------------------- |
| `python-tests.yml`                     | PR — only when `.py`, `.txt`, `.yml`, `.in` changed | Skips on pure translation file PRs                          |
| `validate-translation-files.yml`       | PR                                                  | Validates `.po` + `.json`; posts result as comment          |
| `extract-translation-source-files.yml` | Daily cron (midnight UTC) or manual                 | Clones source repos, extracts strings, opens auto-merged PR |
| `automerge-transifex-app-prs.yml`      | PR from Transifex bot                               | Auto-merges Transifex App translation PRs                   |
| `fix-transifex-resource-names.yml`     | Manual                                              | Renames Transifex resource slugs                            |
| `release-project-sync.yml`             | Manual                                              | Syncs to release Transifex project                          |

## Common Pitfalls

- **Never manually edit source (`/en/`) translation files** — they are extracted by upstream repos and overwritten on the next sync.
- **Never edit `.po` or `.json` translation files directly for content** — content edits belong in the Transifex UI.
- **Python version**: Scripts and CI use **Python 3.12**. Use `python3.12` or a matching virtualenv.
- **`gettext` must be installed** (`msgfmt` command) for `.po` validation to work. On macOS: `brew install gettext`.
- **Node.js**: Use the version specified in `.nvmrc`. Run `npm clean-install` (not `npm install`) to get `@formatjs/cli` for JSON validation.
- **Transifex API token**: Set `TRANSIFEX_API_TOKEN` in your environment before running any Transifex scripts.
- **Requirements pinning**: Always use `make upgrade` to recompile requirements. Never hand-edit `requirements/*.txt` files.
- **Commit messages**: Conventional commit format is enforced by `commitlint.yml` (e.g. `fix:`, `feat:`, `chore:`).

## Dependencies

- Python: `transifex-python`, `edx-i18n-tools`, `python-slugify`, `pyyaml`, `requests`
- Test: `pytest`, `pytest-cov`, `responses`
- Node.js: `@formatjs/cli`
- System: `gettext` (for `msgfmt`), `gh` CLI (for PR retry script)
