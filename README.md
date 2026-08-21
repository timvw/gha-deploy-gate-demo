# Deploy-gate UX demo

Two ways to put a manual approval gate in front of a `test` deploy in
GitHub Actions. Nothing here actually deploys anything — the jobs just
sleep and echo.

## The problem

Say you want a **passive gate**: the pipeline offers a deploy to test,
and somebody approves it when they're ready. Not a "deploy now" button —
the deploy should be sitting there, waiting.

The obvious way to build that is an `environment:` with required
reviewers on the deploy job. It works, but it has a sharp edge:

> A run parked on an approval gate still counts as **in progress**.

So if the workflow also has `cancel-in-progress: true`, the next push to
`main` cancels the parked run. The pending deploy is silently discarded,
and the run renders grey (`cancelled`) rather than red — no failure, no
notification, nothing that reads as "a deploy was lost".

At steady state you end up with a chain where every run's end time is
exactly the next run's start time, and the deploy never happens.

## A — inline gate

[`a-inline-gate.yml`](.github/workflows/a-inline-gate.yml) — one
workflow, `ci -> deploy-dev -> deploy-test`, with `deploy-test` gated on
the `test-inline` environment, and **workflow-level** concurrency:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Push twice and watch the first run: it's cancelled, and the cancellation
takes `deploy-dev` with it — mid-apply, if the timing is unlucky. That's
the part that can leave a Terraform state lock behind.

## B — split gate

Two workflows:

- [`b-ci-and-dev.yml`](.github/workflows/b-ci-and-dev.yml) —
  `ci -> deploy-dev`, then it's **done**. It always completes, is never
  held open waiting for a human, so it can never be cancelled while
  parked. Concurrency is per-job: `ci` supersedes freely, `deploy-dev`
  **queues** (`cancel-in-progress: false`) so an in-flight apply is never
  interrupted.
- [`b-deploy-to-test.yml`](.github/workflows/b-deploy-to-test.yml) —
  triggered by `workflow_run` once the above succeeds. Holds only the
  gate. Concurrency covers only the deploy, so a newer commit supersedes
  a pending approval — you never want to approve a stale SHA — without
  touching `ci` or `deploy-dev`.

Same approval UX. But the thing sitting in `waiting` is now a run called
**"B - deploy to test"**, instead of a grey cancelled run titled after
whatever feature branch happened to merge.

## Try it

```bash
git commit --allow-empty -m "commit one" && git push
# wait ~30s, then while deploy-dev is still running:
git commit --allow-empty -m "commit two" && git push
```

Then open the Actions tab:

| | A (inline) | B (split) |
|---|---|---|
| first run | `cancelled` — `deploy-dev` killed mid-apply | ci+dev **completes**; deploy run superseded |
| second run | parks on the gate | ci+dev completes; deploy run parks |
| what's in `waiting` | a run named after a feature branch | a run named "deploy to test" |
| CI pipeline blocked? | yes, until somebody clicks | no |

Approve from the run page, or via **Environments** in the repo sidebar.

## Caveats

- `workflow_run` triggers use the copy of the workflow file on the
  **default branch**, so edits to `b-deploy-to-test.yml` only take effect
  once merged to `main`. Annoying while iterating, irrelevant afterwards.
- Environment protection rules are free on public repos, but need
  Pro/Team/Enterprise on private ones. This repo is public so the gate
  actually works.
