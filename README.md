# Deploy-gate UX demo

Two ways to put a manual approval gate in front of a `test` deploy.
Nothing here actually deploys anything — the jobs just sleep and echo.

## The question

We want a **passive gate**: the pipeline offers a deploy to test, and
somebody approves it when they're ready. Not a "deploy now" button.

The problem with how we do that today is that the run sits in `waiting`
for hours or days, and a run parked on an approval still counts as
*in progress* — so the next push to `main` cancels it. The deploy is
silently lost, and the run renders grey (`cancelled`), not red.

## A — inline gate (how it works today)

`a-inline-gate.yml` — one workflow, `ci -> deploy-dev -> deploy-test`,
with `deploy-test` gated on the `test-inline` environment, and
**workflow-level** concurrency:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Push twice in a row and watch the first run: it is cancelled, and the
cancellation takes `deploy-dev` with it — mid-`apply`, if the timing is
unlucky. That's the bit that risks a Terraform state lock in the real
pipeline.

## B — split gate (proposed)

Two workflows:

- `b-ci-and-dev.yml` — `ci -> deploy-dev`, then it's **done**. Always
  completes, is never held open by a human, so it can never be cancelled
  while parked. Concurrency is per-job: `ci` supersedes freely,
  `deploy-dev` **queues** (`cancel-in-progress: false`) so an in-flight
  apply is never interrupted.
- `b-deploy-to-test.yml` — triggered by `workflow_run` when the above
  succeeds. Holds only the gate. Concurrency covers only the deploy, so a
  newer commit supersedes a pending approval without touching anything
  else.

Same approval UX. But the thing sitting in `waiting` is now a run called
**"B - deploy to test"** instead of a grey cancelled run titled after
some unrelated feature branch.

## Try it

```bash
git commit --allow-empty -m "commit one" && git push
# wait ~30s, then while deploy-dev is still running:
git commit --allow-empty -m "commit two" && git push
```

Then open the Actions tab:

| | A (inline) | B (split) |
|---|---|---|
| first run | `cancelled` — deploy-dev killed mid-apply | ci+dev **completes**; deploy run superseded |
| second run | parks on gate | ci+dev completes; deploy run parks |
| what's in `waiting` | a run named after a feature branch | a run named "deploy to test" |
| pipeline blocked? | yes, for as long as nobody clicks | no |

Approve from the run page, or under **Environments** in the repo sidebar.

## Caveat

`workflow_run` triggers use the copy of the workflow file on the
**default branch**, so edits to `b-deploy-to-test.yml` only take effect
once merged to `main`. Mildly annoying while iterating, irrelevant in
steady state.
