# choose-runner

Reusable GitHub Actions workflow that picks where a job runs: the first
self-hosted runner label in a priority list that has an **online** runner,
else a GitHub-hosted fallback.

Default order: `persephone` → `hestia` → `ubuntu-latest`.

## Usage

```yaml
jobs:
  choose-runner:
    uses: claymaks/choose-runner/.github/workflows/choose.yml@v1
    secrets:
      RUNNER_STATUS_PAT: ${{ secrets.RUNNER_STATUS_PAT }}
    # with:
    #   order: persephone,hestia     # most preferred first
    #   fallback: ubuntu-latest      # label or JSON array
    #   extra_labels: dind           # runner must carry these too

  build:
    needs: choose-runner
    runs-on: ${{ fromJSON(needs.choose-runner.outputs.runner) }}
    timeout-minutes: 60
    steps:
      - run: echo hello
```

The `runner` output is JSON, so always wrap it in `fromJSON()`:

- a self-hosted match: `["self-hosted","persephone","dind"]`
- no match: `"ubuntu-latest"` (or the `fallback` you passed)

## Inputs

| Input | Default | Meaning |
|-------|---------|---------|
| `order` | `persephone,hestia` | Labels to try, most preferred first |
| `fallback` | `ubuntu-latest` | `runs-on` when nothing in `order` is online |
| `extra_labels` | `dind` | Labels the runner must also carry; added to `runs-on` |

## How it decides

It lists the **calling** repo's self-hosted runners
(`GET /repos/{owner}/{repo}/actions/runners`) and, for each label in
`order`, counts runners that are `online` and carry that label plus every
`extra_labels` label. The first label with at least one wins. Matching is
by label, never by runner name, because ephemeral runners get a new random
name every job.

Online is not idle: if every matching runner is busy, the job queues for
that label instead of falling through. Missing PAT or an API error means
the fallback.

## The PAT

`GITHUB_TOKEN` cannot list self-hosted runners (403). Pass a fine-grained
PAT as `RUNNER_STATUS_PAT` with **Administration: read** on the calling
repo (the repo whose runners you want to see).

Pass the secret explicitly as shown. `secrets: inherit` only works when the
caller is in the same org/owner as this repo.

This repo is public so repos under any owner can call it; a private
reusable workflow is only callable within its own owner.

## Versioning

Callers pin `@v1`. Compatible fixes move the `v1` tag; breaking changes get
`v2`.
