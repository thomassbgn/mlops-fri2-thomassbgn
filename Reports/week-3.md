# Week 3, Labs 5–6: deployment, structured logs, model-version checks, GitHub Actions

## Student / team

- Student: Thomas Soubagne (GitHub `thomassbgn`), session fri2
- Team: "vant" (Nils Giraud, Victor Rouleau, Axel Alacache, Thomas Soubagne)
- Team repository: [BaptVic78/mlops-fri2-vant-bike-demand](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand)
- Work split: `WORK-SPLIT-LABS-5-8.md` in the team repository.

| Lab | My part | Reviewed by | I reviewed |
|---|---|---|---|
| Lab 5 | README `Deploy` section (the ten questions) and the Lab 5 team report | Axel Alacache | Nils's request logs (PR #16) |
| Lab 6 | Container image in CI; planted-defect PR against Victor's tests | Axel (image) / Victor (defect PR) | Nils's image job (PR #21) |

In Lab 6, Nils took over the image job (PR #21), so I reviewed and merged his
version instead of opening my own (see Blockers).

## Links to code and pull requests

### Lab 5

- [PR #16](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/pull/16), Nils's structured request logs. I reviewed it, approved it and merged it as `461c867`.
- [PR #17](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/pull/17), my PR, "Lab 5: README Deploy answers and run evidence". Commits `0cc1806` and `52024be` (review fix). Reviewed and merged by Axel as `3f77fb1`.
  - `README.md`, section `Deploy` to "Deploy: design choices" (the ten questions: kept, rejected, why).
  - [`reports/lab-05.md`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/blob/main/reports/lab-05.md) and [`reports/lab5-evidence/`](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/tree/main/reports/lab5-evidence) (`image-id.txt`, `requests.txt`, `logs.txt`, `failure.txt`).
- [PR #18](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/pull/18), Axel's external check `scripts/check_deployment.py`, merged as `a8e8380`. It is not my work; my README points to it.

### Lab 6

- [PR #21](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/pull/21), Nils's `image` job in `.github/workflows/ci.yml`. I reviewed it, approved it and merged it as `4a5e11e`.
- [PR #22](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/pull/22), my planted-defect PR. Commits `185d15e` (defect), `9e44ad2` (repair) and `bcdba96` (report). Waiting for Victor's review at the time of writing.

## Test and execution results (commands, revision, observed output)

### Lab 5 (local, not CI)

Windows 11, Docker Desktop 28.5.1. Image built from a clean checkout of `main` at `461c867`.

| Step | Command | Observed |
|---|---|---|
| Build | `docker build -t bike-demand:lab5 .` | exit 0, `sha256:76962f6103dab76d24a6b3d2d26705a41ea32cf10cc2367cb8879f166a3f88af` |
| Run | `docker run --rm -d --name bike-lab5 -e UVICORN_PORT=8080 -p 127.0.0.1:8000:8080 bike-demand:lab5` | started; model loaded in 6 s |
| Ready | `curl --fail --retry 60 --retry-all-errors --retry-delay 2 http://127.0.0.1:8000/health` | 200 `{"status":"ok","model_version":"baseline-462260e47aa9"}` |
| Valid | `POST /predict` (README example) | 200, `prediction` 146.595..., same `model_version` |
| Invalid | `POST /predict` with `hr: 24` | 422 `Input should be less than or equal to 23` |
| Logs | `docker logs -t bike-lab5` | one JSON line per request: `"status_code": 200, "outcome": "accepted"` and `"status_code": 422, "outcome": "rejected"`, with `duration_ms` and `model_version` |
| Failure | same image started after `rm artifacts/model.joblib` | `/health` and `/predict` 503 `Model not available: FileNotFoundError ...`; log `"outcome": "error", "model_version": null`; after `docker stop` nothing left, image unchanged |

Local checks on my branch: `ruff check` passed, `ruff format --check` passed, `pytest -q` 149 passed.
After Axel's review I reran the README commands in PowerShell 5.1 with `curl.exe`: health 200, predict 200, broken health 503.

### Lab 6 (GitHub Actions runs, not my laptop)

PR #22, the same pull request throughout:

| Commit | Run | Result |
|---|---|---|
| `185d15e` defect | [42511172980](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/actions/runs/42511172980) | lock and style passed, tests failed, image skipped. `E assert 143.79 == 143.79444765760556`, `1 failed, 159 passed` |
| `9e44ad2` repair | [42511337831](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/actions/runs/42511337831) | all four jobs passed, `160 passed, 2 warnings in 14.24s`, image `sha256:46cad523` |

The raw red log is saved as text in the team repository: `reports/lab6-evidence/fault-pr-red.txt`.

PR #21 (Nils), which I reviewed: run [42364256011](https://github.com/BaptVic78/mlops-fri2-vant-bike-demand/actions/runs/42364256011) was green. The `image` job built `sha256:67ee7667`. The packaged `model_sha256` was `d24c050e`, the same as my local Lab 5 image.

## Screenshots

TODO before submission, 2 to 4 images in `Reports/images/`:

1. `week-3-pr22-red.png`: PR #22 checks on `185d15e`, `tests` red.
2. `week-3-pr22-green.png`: PR #22 checks on `9e44ad2`, all green.
3. `week-3-lab5-logs.png`: terminal with `docker logs` showing the accepted and the rejected request.

## My contribution

### Lab 5

- I reviewed Nils's logging PR #16 by running it: I built the image and sent one valid and one invalid request. I checked that the two log lines were readable and that no body or header was logged. Then I approved and merged it. Non-blocking remarks I left: a 500 no longer logs its cause, there is no test of the log line, and the `Dockerfile` lost its final newline.
- I wrote the README `Deploy: design choices` table: settings, model, ready, broken, log line, secrets, identity, expected, promise, failure. Each answer has the kept option, the rejected option and the reason.
- I ran the deliberate failure (missing model) and recorded what it leaves behind.
- I wrote `reports/lab-05.md`, linking commit, then image ID and command, then requests and log lines. The raw outputs are in `reports/lab5-evidence/`.
- I found and fixed a problem in the startup instructions (see Decisions).

### Lab 6

- I chose the planted defect myself and did not tell Victor before the PR. It was a "cosmetic" rounding of the training mean in `compare_mean`. I opened PR #22, saw CI turn red, read the cause in the log, pushed the repair on the same PR, saw it green, and recorded everything in `reports/lab-06.md`.
- I reviewed PR #21 (image job) from the runner logs, then approved and merged it.

## Decisions and rationale

- Readiness wait (Lab 5). The README's `curl --retry 15 --retry-connrefused` failed on my machine with `curl: (52) Empty reply from server`. While the model loads (6 to 40 s), Docker Desktop accepts the connection and then closes it, and `--retry-connrefused` does not retry that. I kept `--retry 60 --retry-all-errors --retry-delay 2`. I rejected a fixed `sleep`, because it is either too short or wasted time. `--retry-all-errors` also retries a 503, so the wait still fails if the model never loads.
- Which failure to cause (Lab 5). I chose a missing model, started from the same image with `--entrypoint sh -c "rm ..."`. I rejected rebuilding a broken image. This failure tests the README "broken" answer directly, and the image stays unchanged.
- Which defect to plant (Lab 6). I chose rounding the baseline mean. It looks harmless in review, but it silently changes the recorded comparator. I rejected an obvious bug such as a syntax error or a wrong feature name, because many tests would catch it and it would say nothing about Victor's regression test.

## Review and response

- Axel on my PR #17: "On Windows PowerShell `curl` is `Invoke-WebRequest` and rejects `--fail` ... please use `curl.exe`."
  - I changed the commands to `curl.exe` in `powershell` blocks and kept POSIX `curl` lines for Git Bash, macOS and Linux.
  - Rerunning in PowerShell 5.1 showed a second bug: single-quoted JSON loses its inner quotes and `/predict` returned 422. I escaped the quotes (`\"`), which returns 200.
  - I added `Start-Sleep 15` before the failure check, which had hit `(52) Empty reply`.
  - I replied on the PR with these points. Axel merged it.
- My review of PR #16 (Nils): approved after running it, with three non-blocking remarks (see My contribution).
- My review of PR #21 (Nils): approved after reading the runner logs. Remarks: save the runner log as text, the image is built but its tests are not run inside it, and the image-job owner differs from `WORK-SPLIT-LABS-5-8.md`.

## Blockers / missing evidence

- Push refused for a workflow file (Lab 6). My own image job (`.github/workflows/image.yml`, local commit `890a01f`) could not be pushed: `refusing to allow an OAuth App to create or update workflow ... without workflow scope`. My Git credential has the `repo` scope but not `workflow`. Meanwhile Nils opened PR #21 with the same job, so I reviewed and merged his instead of duplicating it. My branch was never pushed. To push workflow files later, I need to sign in to Git again with the `workflow` scope.
- PR #22 still needs Victor's review and merge.
- Nils's README CI and Cloud Run PR (which I review) was not open yet when I wrote this report, so the Cloud Run deployment is not part of my evidence.
- The `m3` tag is not created yet. It comes after Labs 5 and 6 are reviewed.
- Screenshots: see the TODO above.

## Assistance and recovery acknowledgement

I used Claude Code (AI assistant) throughout Labs 5 and 6. It read the lab PDFs and the work split, and ran the local Docker runs and the PowerShell reruns. It drafted the README `Deploy` table, `reports/lab-05.md`, the Lab 6 report section and this report. It used the GitHub API to read and post PR reviews and comments and to download runner logs. It also prepared the planted-defect commits. I chose the scope, approved each step, and checked the outputs and runs cited here. The logging code is Nils Giraud's (PR #16), the external check is Axel Alacache's (PR #18), the image job is Nils's (PR #21), and the regression tests are Victor Rouleau's (PR #20). The course starter code was supplied by the instructor.
