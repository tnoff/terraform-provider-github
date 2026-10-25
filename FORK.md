# About this fork

`tnoff/terraform-provider-github` is a temporary fork of
[`integrations/terraform-provider-github`](https://github.com/integrations/terraform-provider-github).
It is **not** a general-purpose distribution; use the upstream provider unless
you are the fleet it was made for.

## Why it exists

Upstream v6.13.0 has no resource for the "Fork pull request workflows from
outside collaborators" policy (issue
[#3445](https://github.com/integrations/terraform-provider-github/issues/3445)).
The implementation is in
[#3446](https://github.com/integrations/terraform-provider-github/pull/3446),
unreleased. This fork is upstream **v6.13.0** plus that commit, so the fleet's
repos can manage the policy from terraform now:

- `github_actions_repository_fork_pr_contributor_approval` (+ data source)
- `github_actions_organization_fork_pr_contributor_approval` (+ data source)

## What differs from upstream

| Change | Why |
|---|---|
| Cherry-pick of #3446 (`a4210779`) | the feature |
| `go-github/v86` -> `v88` imports in the two new resources | #3446 predates v6.13.0's move to v88; import path only |
| `main` is v6.13.0 + the above, **not** upstream's `main` | upstream `main` is unreleased work; we ship a tagged base |
| Trimmed `.goreleaser.yml`: linux/darwin x amd64/arm64, no GPG/cosign/SBOM | upstream's signing keys are not available here |
| Own `ci.yaml`, `release.yaml`, `upstream-watch.yml`, `notify-failure.yml` | fleet conventions; upstream's acceptance-test, labeler, stale and bot workflows removed |
| `CODEOWNERS` is `* @tnoff`; issue/PR templates and Dependabot config removed | this is not an upstream contribution channel |

## Workflows

- **CI** (`ci.yaml`): build, vet, unit tests, secret scan, workflow-contract
  check; the `CI result` job aggregates them and is the one required check.
  Acceptance tests (`TestAcc*`) are excluded by name, as in upstream's `make test`:
  they need a live org and token.
- **Release** (`release.yaml`): pushing a `v*` tag builds and publishes the zips,
  a `SHA256SUMS` file and a build-provenance attestation. A manual run
  (`workflow_dispatch`) does a snapshot build and uploads `dist/` as a workflow
  artifact without publishing.
- **Upstream watch** (`upstream-watch.yml`, weekly): fails, and so pings Discord via
  `notify-failure.yml`, in two cases: an upstream release now contains the resource
  (retire the fork), or upstream's latest release is newer than `.fork-base` (we
  are out of date; rebase). The summary lists the files both sides changed, which
  is where conflicts will be.
- **Scheduled** (`scheduled.yml`, weekly): Renovate, limited by `renovate.json` to
  the workflow pins. Go modules are deliberately not managed, so the code stays
  "upstream's release plus our patch"; upstream's own dependency bumps arrive with
  the next upstream release. Nothing automerges. Needs the `CI_APP_*` secrets and
  the tnoff-ci App on this repo.
- **Notify on failure**: fleet-wide Discord alert for non-PR failures.

## Verifying a release

The zips are not GPG-signed. Check provenance with the GitHub CLI:

```bash
gh attestation verify terraform-provider-github_<version>_linux_arm64.zip --repo tnoff/terraform-provider-github
```

## Rebasing onto a new upstream release

`.fork-base` holds the upstream tag this fork is based on. `main` is that tag
plus our commits (the #3446 patch, the import fix, and fork-only CI/docs), so a
new base is a rebase of those commits:

```bash
git remote add upstream https://github.com/integrations/terraform-provider-github.git   # once
git fetch upstream --tags
git checkout -b rebase-v6.14.0 main
git rebase --onto v6.14.0 "$(cat .fork-base)" rebase-v6.14.0
echo v6.14.0 > .fork-base && git commit -am "chore: rebase onto upstream v6.14.0"
go build ./... && go vet ./... && go test ./...
```

Expect conflicts in `github/provider.go` (the four registrations) and anywhere
`upstream-watch` listed. If upstream renamed or changed the go-github major
version, fix the import in the two new resources as `2d595a96` did. Open a PR
into `main`, then tag a release.

## Version numbers

Tag releases `v<upstream base>+fork.<n>`, for example `v6.13.0+fork.1`, then
`v6.13.0+fork.2` for a fork-only change, and `v6.14.0+fork.1` after rebasing onto
upstream v6.14.0. The `+fork.N` part is semver *build metadata*.

Tested against a Terraform filesystem mirror (2026-10-25), because the obvious
alternatives fail:

| Version | `~> 6.0` | Exact pin |
|---|---|---|
| `6.13.0+fork.1` (this scheme) | selected | `= 6.13.0` does **not** match it |
| `6.13.0-fork.1` (pre-release suffix) | **never selected** | only `= 6.13.0-fork.1` |
| `fork-6.13.0` (prefix) | not valid semver | n/a |

So consumers must use range constraints (the fleet uses `~> 6.0`); an exact pin of
the plain upstream version will not pick the fork up. `terraform version` and the
lock file show `+fork.N`, so a patched build is never mistaken for the registry's.

GitHub may rewrite `+` in release asset names. Consumers must therefore derive the
mirror file name from the *tag* (`terraform-provider-github_<version>_<os>_<arch>.zip`)
and rename on download, not trust the asset name.

## Retiring the fork

When `upstream-watch` fails (or you see the resource in an upstream release):

1. Point consumers back at `integrations/github` at that release and drop the
   mirror configuration.
2. Remove the fork's block and `import`s from `tnoff/terraform` (`infra/repos.tf`)
   and archive or delete this repository.
