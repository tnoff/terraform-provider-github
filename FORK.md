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
  Acceptance tests (`TestAcc*`) skip without `TF_ACC` and are not run.
- **Release** (`release.yaml`): pushing a `v*` tag builds and publishes the zips,
  a `SHA256SUMS` file and a build-provenance attestation. A manual run
  (`workflow_dispatch`) does a snapshot build and uploads `dist/` as a workflow
  artifact without publishing.
- **Upstream watch** (`upstream-watch.yml`, weekly): fails, and so pings Discord via
  `notify-failure.yml`, when an upstream release contains the resource.
- **Notify on failure**: fleet-wide Discord alert for non-PR failures.

## Verifying a release

The zips are not GPG-signed. Check provenance with the GitHub CLI:

```bash
gh attestation verify terraform-provider-github_<version>_linux_arm64.zip --repo tnoff/terraform-provider-github
```

## Version numbers

Tag releases `vX.Y.Z` with a plain version and no pre-release suffix. Terraform
does not select pre-release versions for range constraints such as `~> 6.0`, so a
tag like `v6.13.0-fork.1` would never be picked.

## Retiring the fork

When `upstream-watch` fails (or you see the resource in an upstream release):

1. Point consumers back at `integrations/github` at that release and drop the
   mirror configuration.
2. Remove the fork's block and `import`s from `tnoff/terraform` (`infra/repos.tf`)
   and archive or delete this repository.
