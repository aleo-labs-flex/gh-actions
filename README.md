# gh-actions

The reusable workflow and composite actions that aleo-labs-flex repositories
use to start and run on the self-hosted runner fleet. The fleet itself — the
boxes, their provisioning and the AWS role this workflow assumes — is managed
in the private `aleo-labs-flex/aws-dev-infra`.

This repository is public so that public repositories can call it: GitHub does
not let a public repository use actions from a private one. Nothing here is
secret. What decides who can start a box is the AWS role's trust policy, which
admits named caller repositories only.

## Using it
`gpu-test.yml` in snarkVM-cuda and dps both need a GPU runner started, and that
logic was 128 identical lines in each before it was shared. It is published as a
reusable workflow plus composite actions, so a fix lands once.

```yaml
jobs:
  # ONE start job. It sizes the whole fleet and starts what is missing, so there
  # is no second caller to keep in step with this one. actions:read is what lets
  # it count the queue; secrets: inherit carries the runner-census key.
  start-runners:
    uses: aleo-labs-flex/gh-actions/.github/workflows/start-runner.yml@v1
    permissions: { id-token: write, contents: read, actions: read }
    secrets: inherit
    with:
      role-to-assume: ${{ vars.RUNNER_ROLE_ARN }}
      runner-app-id: ${{ vars.RUNNER_APP_ID }}
    # Rejects pull_request_target outright, then admits non-PR triggers and
    # same-repo PRs. Note the first clause: `event_name != 'pull_request'` is
    # TRUE for a pull_request_target event, so an `||` guard short-circuits and
    # lets it through -- on the one trigger where the token is writable.
    if: >-
      github.event_name != 'pull_request_target' &&
      (github.event.pull_request == null ||
       github.event.pull_request.head.repo.full_name == github.repository)

  gpu-test:
    needs: start-runners
    runs-on: [self-hosted, linux, x64, gpu, sm_89]
    steps:
      - uses: aleo-labs-flex/gh-actions/.github/actions/gpu-runner-preflight@v1
      - uses: actions/checkout@v5
        with: { persist-credentials: false }
      - run: cargo test --release --features cuda      # your own command
      - uses: aleo-labs-flex/gh-actions/.github/actions/gpu-runner-report@v1
        if: always()
```

The split is by shape, not by taste. `start-runner` is an entire job, so it is a
reusable workflow — the caller cannot set `runs-on` for a composite action.
Preflight and report are steps inside a job the caller must own, because the
test command differs per repository, so they are composite actions — a reusable
workflow cannot be interleaved with a caller's own steps.

## Giving a caller the fleet-wide view

Optional. Without it a caller sizes the fleet from its own queue alone, which is
smaller than the truth whenever another repository is also building.

The org-wide count needs to ask GitHub which runners are busy, from inside a
job on `ubuntu-latest` — so it needs a credential that reaches a workflow, and
a private key only reaches one through a secret.

**Use a single-purpose GitHub App.** A key in org secrets can mint every
permission its app holds, for any workflow in the org that can read secrets. The
app that administers this fleet must therefore not be the one whose key goes
there: it can write org secrets, and that would be handing that power to CI.

One app, one permission:

| | |
| --- | --- |
| Organization → Self-hosted runners | **Read-only** |
| everything else | untouched — no repository permissions at all |
| Webhook | off |

Create it, **generate a private key**, then **install it** on the org — creating
and installing are separate steps, and an uninstalled app authenticates fine
while resolving no installation, which looks like a broken key and is not.

Then store the two halves:

```sh
gh variable set RUNNER_APP_ID  --org aleo-labs-flex --visibility all --body <app id>
gh secret   set RUNNER_APP_KEY --org aleo-labs-flex --visibility all < path/to/key.pem
```

The id is a **variable**, not a secret: it is not sensitive, and having it in
the log makes a misconfiguration diagnosable instead of mysterious.

A caller then passes both through:

```yaml
  start-cpu-runners:
    uses: aleo-labs-flex/gh-actions/.github/workflows/start-runner.yml@v1
    permissions: { id-token: write, contents: read, actions: read }
    with:
      role-to-assume: ${{ vars.RUNNER_ROLE_ARN }}
      fleet: cpu-runner cpu-runner2 cpu-runner3 cpu-runner4
      label: cpu
      ladder: c6i.8xlarge c6a.8xlarge m6i.8xlarge
      runner-app-id: ${{ vars.RUNNER_APP_ID }}
    secrets:
      runner_app_key: ${{ secrets.RUNNER_APP_KEY }}
```

No thresholds in the caller: how deep a queue justifies another box is a
property of the fleet, and it lives with the fleet. The caller says which boxes
exist and which label they answer to.

**The secret name has underscores on purpose.** GitHub secret names cannot contain
hyphens, so a `runner-app-key` secret could never be supplied by `secrets: inherit`
— it would fall back to per-repo sizing silently, forever, with nothing in any
log to say so.

Everything here fails **open**: no variable, no secret, a key that does not
match, an app installed elsewhere, a rate-limited listing. Each falls back to
`busy = 0` — the fleet is then sized from this repository's queue alone, which
under-provisions rather than over-provisions, and a run is never reddened by a
count it could not take.

## Versioning

Callers pin `@v1`. The AWS role trusts `start-runner.yml` **only at a `v*` tag**
— not a branch, not a SHA — so a caller pinned any other way is denied at STS
with "Not authorized to perform sts:AssumeRoleWithWebIdentity", which names
neither the ref nor the claim. Every caller should run `check-shared-refs`: it
fails on a ref that is not `@vN`, and on any ref to `aws-dev-infra`, where this
code used to live.

A tag ruleset on `v*` restricts creating, **updating** and deleting tags to its
bypass list. Moving `v1` is an update, and it changes what runs with the
fleet's AWS credentials in every caller at once, so move it deliberately.

**Trying a change before it is released** needs a tag too, since the role
trusts nothing else: someone on the bypass list tags the commit `v0-<topic>`,
a caller branch points at that tag, and the tag is deleted afterwards. The
ruleset covers it like any other `v*`, and `check-shared-refs` fails on it, so it
cannot be merged by accident.

The actions `start-runner.yml` uses run holding the role, and the trust policy
names this file, not what it pulls in, so they are pinned to commit SHAs;
Dependabot proposes the updates.

## Development

```sh
./lint-workflow-shell.sh   # shellcheck the shell inside the workflow and action YAML
./test.sh                  # the steps' own shell, run against stubs; no AWS
```
