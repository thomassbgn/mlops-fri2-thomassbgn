# Week 2, Labs 3–4: dependency lock, Docker, clean rerun, tested prediction API

## Student / team

- Student: Thomas Soubagne (GitHub `thomassbgn`), session fri2
- Team: "vant" (Nils Giraud, Victor Rouleau, Axel Alacache, Thomas Soubagne)
- Team repository: [BaptVic78/mlops-fri2-vant-bike-demand](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand)
- This report covers Lab 3 (dependency lock, Docker, clean rerun). Lab 4 is not done yet (see *Blockers / missing evidence*).

## Links to code and pull requests

- Starting point: [PR #7](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/pull/7) by Nils Giraud (first Dockerfile and `.dockerignore`), merged into `main` as `022c3c4`.
- My branch: [`feature/lab3-docker-reproducibility`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/tree/feature/lab3-docker-reproducibility), commit [`30d5c3e`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/commit/30d5c3e708459cd4fb5ccb56acaba83d91e9be63) on top of `022c3c4`.
- Files at that commit:
  - [`Dockerfile`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/blob/30d5c3e708459cd4fb5ccb56acaba83d91e9be63/Dockerfile)
  - [`.dockerignore`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/blob/30d5c3e708459cd4fb5ccb56acaba83d91e9be63/.dockerignore)
  - [`README.md`, section "Lab 3 – Reproducible Docker Environment"](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/blob/30d5c3e708459cd4fb5ccb56acaba83d91e9be63/README.md)
  - [`reports/lab-03.md`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/blob/30d5c3e708459cd4fb5ccb56acaba83d91e9be63/reports/lab-03.md) (execution record)
  - [`reports/lab3-evidence/`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/tree/30d5c3e708459cd4fb5ccb56acaba83d91e9be63/reports/lab3-evidence) (raw logs `00` to `08`)
- My Lab 3 PR: **TODO: PR link + merge commit SHA**

## Test and execution results (commands, revision, observed output)

All commands below were run **locally** on my Windows 11 machine with Docker Desktop (Docker 28.5.1, linux/amd64 engine). None of them is a GitHub Actions / CI result; the CI workflow was not changed and does not build the image.

**Revision.** The checks ran on `022c3c4` plus my Lab 3 changes (uncommitted at the time of the runs), then committed as `30d5c3e`. Host: uv 0.12.24, Python 3.12.15 (uv-managed).

**Image.** `bike-demand:lab3`, `sha256:af0942bc3c4cd4096959afe0e960f51428d12303026ff45e87915367eb302501`, built 2026-10-22T07:14:33Z with `--no-cache`, 1.54 GB. Inside the image: Python 3.12.12, uv 0.12.24, git 2.47.3, scikit-learn 1.9.1, pandas 2.3.3, numpy 2.5.3, mlflow 3.17.0, joblib 1.6.0 (same library versions as the host `.venv`).

**Lock.** `uv.lock` was not modified (98 packages resolved). I never ran `uv lock`.

| # | Command | Observed result | Log |
| --- | --- | --- | --- |
| 1 | `uv lock --check` | exit 0 | `01` |
| 2 | `uv sync --locked` | exit 0 | `01` |
| 3 | `uv run --locked pytest -q` | 49 passed, 1 warning | `01` |
| 4 | `uv run --locked ruff check .` / `ruff format --check .` | pass / pass | `01` |
| 5 | `docker build --no-cache --progress=plain -t bike-demand:lab3 .` | exit 0; `Installed 94 packages` from the lock | `02` |
| 6 | `docker run --rm bike-demand:lab3` | exit 0 | `03` |
| 7 | `docker run --rm --network none bike-demand:lab3` | exit 0 | `04` |
| 8 | `docker run --rm --network none bike-demand:lab3 python -m bike_demand.preflight` | `Preflight passed` | `05` |
| 9 | `docker run --rm --network none bike-demand:lab3 pytest -q` | 49 passed, 1 warning | `05` |
| 10 | `docker run --rm --network none bike-demand:lab3 ruff check .` | `All checks passed!` | `05` |
| 11 | `docker run --rm --network none bike-demand:lab3 uv lock --check --offline` | exit 0 | `05` |

The single pytest warning is an MLflow/SQLAlchemy `SADeprecationWarning`, the same on the host and in the image. In runs 6 and 7, stderr only contains MLflow INFO lines: no build, install or resolve line at container start.

**Reproducibility: Lab 2 vs Docker** (copied from `reports/lab-03.md`, checked against `03-docker-run.stdout.json` and `04-docker-run-no-network.stdout.json`):

| Field | Lab 2 reference (committed `reports/metrics.json`) | Host rerun, clean copy of `022c3c4` (Windows) | Docker (network) | Docker (`--network none`) |
| --- | --- | --- | --- | --- |
| Validation MAE, random forest | 86.84685685591091 | 86.84685685591091 | 86.84685685591091 | 86.84685685591091 |
| Validation MAE, naive train mean | 154.23179306864967 | 154.23179306864967 | 154.23179306864967 | 154.23179306864967 |
| `train_mean` | 143.79444765760556 | same | same | same |
| `data_sha256` | `e03de4ee…504f` | same | same | same |
| Partitions (rows) | 8645 / 4358 / 4376 | same | same | same |
| Config / effective RF settings | n_estimators 50, max_depth 10, random_state 42, n_jobs 1 | same | same | same |
| Readback (reload MAE delta, max abs prediction delta) | 0.0 / 0.0 | 0.0 / 0.0 | 0.0 / 0.0 | 0.0 / 0.0 |
| `model_sha256` | `d24c050e…8c07` | `fd98ceaf…304` | `d24c050e…8c07` | `d24c050e…8c07` |
| `training_commit` | `c4ba1825…` (dirty) | `unavailable` (no `.git` in copy) | `unavailable` | `unavailable` |
| `model_run_id` | `ba0ece8cf43541edb4da633481c1f3c5` | `0989a2247a894593a9282b713e4aa8de` | `5a9e754e417a48e586cebc1c719d664f` | `1f7af4b38c494eb6b0885555c94c8c12` |
| `naive_run_id` | `9226418acc914919b61a5640300d6da7` | `c287562efd8a4f2695940aeaf8690470` | `15ff596929eb4c33849eb5f6b0263294` | `6b575c742e21430890ad389b171c32a6` |

The metrics are bit-for-bit identical to Lab 2. Run IDs are random per run, so they differ by design. Training inside the container also wrote `reports/metrics.json`, `reports/metrics-history.jsonl` (2 lines) and `artifacts/model.joblib` (log `06`).

**Negative lock tests** (log `07`, run on a temporary `git archive` copy of `022c3c4`, deleted afterwards):

1. `uv.lock` absent:
   - `uv sync --locked` → `` error: Unable to find lockfile at `uv.lock`, but `--locked` was provided. ``, exit 1, no lock created.
   - `docker build` → fails at `COPY` with `"/uv.lock": not found`, exit 1.
2. `uv.lock` stale (`"tabulate>=0.9,<1"` added to `pyproject.toml` only):
   - `uv lock --check` → `` error: The lockfile at `uv.lock` needs to be updated, but `--check` was provided. ``, exit 1.
   - `uv sync --locked` → `` error: The lockfile at `uv.lock` needs to be updated, but `--locked` was provided. ``, exit 1.
   - `docker build` → fails at `RUN uv sync --locked --no-install-project` with the same error, `exit code: 1`.

In both cases the build stops instead of silently resolving new versions.

## Screenshots

![Successful docker build of bike-demand:lab3 with its image ID](images/week-2-docker-build.png)

`docker build --no-cache` of `bike-demand:lab3` succeeding, image `sha256:af0942bc3c4c…`, branch `feature/lab3-docker-reproducibility` (`30d5c3e`).

![Offline training run in the container showing the validation MAE](images/week-2-docker-run-offline.png)

`docker run --rm --network none bike-demand:lab3`: validation MAE 86.84685685591091 (random forest) and 154.23179306864967 (naive), image `sha256:af0942bc3c4c…`.

![pytest inside the image: 49 passed](images/week-2-docker-pytest.png)

`docker run --rm --network none bike-demand:lab3 pytest -q`: 49 passed, 1 warning, image `sha256:af0942bc3c4c…`.

![uv sync --locked failing on a stale lock](images/week-2-stale-lock-error.png)

Negative test on a temporary copy of `022c3c4` with `tabulate` added to `pyproject.toml` only: `uv sync --locked` refuses the stale `uv.lock` and exits with code 1.

## My contribution

**Audit of the repository.** My local copy of the team repository was outdated (`a44e591`, without the Lab 2 merge from PR #6 or PR #7). I restarted from `origin/main` at `022c3c4` on a new branch, `feature/lab3-docker-reproducibility`.

**Audit of the PR #7 Dockerfile and `.dockerignore`** (by Nils Giraud). I built and ran the PR #7 version and recorded the evidence in log `00`. I found these issues:

| Observation | Consequence |
| --- | --- |
| `.dockerignore` patterns such as `.venv/` and `__pycache__/` only match the root | The untracked `mlops-course/` folder (607 MB, with its own Windows `.venv`) and host `__pycache__` folders were sent; build context of 591 MB |
| `COPY . .` | Everything not ignored entered the image |
| `CMD ["uv", "run", "--locked", ...]` | Every container rebuilt and reinstalled the project at start (`Building bike-demand @ file:///app`, `Uninstalled 1 package` / `Installed 1 package`), even with `--network none` |
| No `UV_PYTHON_DOWNLOADS=never` | uv was allowed to download another interpreter |
| uv cache kept in the image | Image of 2.16 GB |

**My changes:**

- `CMD` runs `/app/.venv/bin/python` directly (via `PATH`), so there is no `uv` resync, rebuild or network access at runtime.
- `UV_PYTHON_DOWNLOADS=never`, so uv must use the image's Python 3.12.12.
- uv cache as a build-only cache mount (`RUN --mount=type=cache,target=/root/.cache/uv`).
- Targeted `COPY` of `data/`, `configs/`, `README.md`, `ci.yml`, `reports/`, `tests/` and `src/` instead of `COPY . .`.
- `.dockerignore` rewritten with `**/` patterns, and exclusion of `mlops-course/`, lab reports, `reports/lab3-evidence/` and `reports/images/`.
- Result: image of 1.54 GB instead of 2.16 GB.
- README section "Lab 3 – Reproducible Docker Environment", `reports/lab-03.md`, and raw logs in `reports/lab3-evidence/`.

**What I kept from PR #7, and why:**

- `python:3.12.12-slim` and uv `0.12.24`: both pinned, which is what a reproducible build needs.
- `uv sync --locked`: it installs only what is in `uv.lock` and fails if the lock does not match `pyproject.toml`.
- The `git` package: `training_commit()` calls the git binary and raises `OSError` if it is missing.
- The order: manifests and lock copied first, then `uv sync --locked --no-install-project`, so the dependency layer is only rebuilt when they change.

## Decisions and rationale

- **No `uv lock`.** The goal of Lab 3 is to rebuild the environment from the existing lock, not to regenerate it. `uv.lock` is unchanged and `uv lock --check` passes on the host and inside the image.
- **No change to data, split, features or hyperparameters.** No tuning and no test-set scoring. This is what makes the MAE comparison with Lab 2 meaningful, and the MAE is identical.
- **Model hash differs between Windows and Linux at equal MAE.** `model_sha256` is `d24c050e…8c07` in both Linux container runs (same as the Lab 2 reference) but `fd98ceaf…304` on the Windows host rerun, which matches an earlier Windows Lab 2 run (`8da42b7`). Predictions and MAE are identical; only the serialized pickle differs. I do not claim byte-identical models across operating systems.
- **`training_commit` is `unavailable` in the container.** `.git` is excluded from the build context, so the code identity is the image ID plus the branch commit.
- **`artifacts/` and `tracking/` excluded from the image.** They are local outputs regenerated by the pipeline, and `tracking/mlflow.db` contains host paths.

**Problems met and how I handled them:**

- Docker Desktop was not running at first (`dockerDesktopLinuxEngine` pipe not found). I started it and reran the Docker steps.
- A host rerun in a deep temp directory failed with `ModuleNotFoundError: joblib.externals.loky.backend.stdlib_py314_resource_tracker` because the path exceeded the Windows 260-character limit. The rerun in a short path succeeded. This is a host limitation, not a project defect.
- In log `07`, the line "lock unchanged … NO" was a false alarm: it compared the LF lock in the temp copy with the CRLF lock in my working tree (`core.autocrlf=true`). I added a note to the log: both SHA-256 values equal the git blob (`8598f1f2…0330`), and `git status` shows the real `uv.lock` unmodified.
- The first image (`9a5f6f11a2c2`) also contained `reports/lab-*.md` and `reports/images/`, so editing the report would have changed the image it documents. I added those paths to `.dockerignore`, rebuilt with `--no-cache`, and regenerated logs 02–06 and 08 on the final files (same metrics as the first image).

## Review and response

TODO: reviewer comment and my response

## Blockers / missing evidence

- **Lab 4 (tested prediction API) is not done yet.** There is no Lab 4 code, test or evidence in this report.
- My Lab 3 PR link, review and merge commit SHA are still missing (see above).
- Lab 3 was only verified locally. It was not verified in GitHub Actions; the CI workflow is unchanged and does not build the image.
- Only linux/amd64 was tested. The base images and the Debian `git` package are not pinned by digest in the Dockerfile (the digests resolved at build time are recorded in `lab-03.md`).
- The container runs as root, and outputs disappear with `--rm` unless copied out.
- Screenshots are still to be added in `reports/images/`.

## Assistance and recovery acknowledgement

I used Claude Code (AI assistant) to help audit the repository, diagnose the Docker runtime rebuild, and draft the Dockerfile/`.dockerignore` changes, the README section and `lab-03.md`. I reviewed every change, re-ran the commands and checked the logs myself. The PR #7 Dockerfile by Nils Giraud was the starting point for my changes. The course starter code was supplied by the instructor.
