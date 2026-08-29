# Mini-project Work — Mini-project {{MP NO}}, ({{COHORT}})

It's where the learner does the mini-project works.

> This is **practice**, not portfolio. Your individual capstones live in *your own* `portfolio-…` repo and are never built here. Keep the two separate: this repo is disposable; your portfolio is forever.

## mini-project-template — Pre-commit & Quality Checks

Quality checks that run automatically before every commit (and in CI). They keep the repo
clean, consistent, and — because this repo is **public** — free of accidentally-committed secrets.

### One-time setup

```bash
pip install pre-commit      # or: uv tool install pre-commit
pre-commit install          # hooks now run on every `git commit`
```

That's it — from now on the checks run automatically when you commit. To also run them at
push time: `pre-commit install --hook-type pre-push`.

### Everyday use

- They run on the files you changed each time you `git commit`.
- If a hook **fixes** something (formatting, trailing whitespace), re-stage and commit again.
- Run everything by hand any time: `pre-commit run --all-files`.
- Skip once in an emergency: `git commit --no-verify` (avoid — CI will still catch it).
- Update hook versions: `pre-commit autoupdate`.

### What runs

| Hook | What it checks |
|------|----------------|
| `trailing-whitespace`, `end-of-file-fixer`, `mixed-line-ending` | Clean line endings and file endings. |
| `check-yaml`, `check-json`, `check-toml` | Config files actually parse. |
| `check-added-large-files` | Blocks files over 5 MB (keep data out of git). |
| `check-merge-conflict`, `check-case-conflict` | No conflict markers; no case-clashing filenames. |
| `check-executables-have-shebangs`, `check-shebang-scripts-are-executable` | Scripts are consistent. |
| `detect-private-key` + **`gitleaks`** | **Secret scanning** — blocks keys/tokens before they reach a public repo. |
| `ruff`, `ruff-format` | Python linting + formatting. |
| **`sqlfluff-lint`** | SQL linting — BigQuery dialect, dbt-aware (understands `ref()`/`source()` with no credentials). |
| `yamllint` | YAML style (tuned for GitHub Actions / dbt / k8s). |
| `markdownlint` | Markdown style (relaxed for technical docs). |
| `terraform_fmt` | Terraform formatting (only when `.tf` files change). |

Config lives in `.pre-commit-config.yaml`, `.sqlfluff`, `.yamllint.yaml`, and `.markdownlint.yaml` —
edit those to tune rules for your work.

### CI backstop

`.github/workflows/checks.yml` runs the **same** hooks on every pull request and push to `main`,
so the checks hold even if someone commits with `--no-verify`. One source of truth
(`.pre-commit-config.yaml`), enforced both locally and remotely.

## This mini-project

- **Brief:** see the relevant mini-project in the course repo (e.g. `mp1-sql-foundations/mini-project/README.md`).
- **Learner (anchor):** `{{LEARNER}}` — runs the mini-project and merges.
- **Learners:** see the week's plan posted in the cohort discussion.

## How the submission works

1. Each member works on a **branch** (`feat/<your-handle>-<thing>`).
2. Open a **PR** into `main`. CI must be green (lint + tests + `terraform validate` + `dbt parse`).
3. A **learning-mate/faculty-trainer reviews** it; the **requester merges**.
4. End of week: the learner tags a demo commit and the cohort does a short walkthrough.

## Rules

- **No secrets, state, or raw data** committed (`.gitignore` covers the usual suspects).
- Any **cloud resources** the pod stands up get **torn down** at the end of the week — especially Composer / GKE / Dataproc.
- Keep it tidy: this repo may be archived after the week.

## Layout

Structure follows the module's brief. Check the README for this mini-project.
