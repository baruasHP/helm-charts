# Release Workflow Working

## Concept/Reason

The release workflow publishes Helm charts from the `release` branch. It rebuilds every chart package, updates the Helm repository index, and commits only the generated `build/` directory.

The published Helm repository path is `https://hamropatro.github.io/helm-charts/build`.

The workflow follows the same local flow as the Makefile: clean `build/`, lint charts, package charts, generate `build/index.yaml`, copy `README.md` into `build/`, then commit the generated files.

## Trigger

The workflow runs on pushes to the `release` branch.

It does not run for changes that only touch:

- `index.yaml`
- `build/**`
- `**/*.md`
- `.github/workflows/release.yaml`

This prevents release loops and avoids chart repo rebuilds for documentation-only changes.

## What `release.yaml` Does

1. Checks out the `release` branch.

   The job uses full Git history so it can commit generated files back to the same branch.

2. Configures Git.

   Commits are authored as `github-actions[bot]`.

3. Installs Helm.

   The workflow installs Helm `v3.14.4`.

4. Rebuilds the chart repository.

   The workflow removes `build/`, recreates it, lints every chart under `charts/*`, packages every chart into `build/`, creates `build/index.yaml`, and copies `README.md` into `build/`.

5. Commits build output.

   If `build/` changed, the workflow commits it with `Publish Helm chart repo`, rebases onto the latest `release`, and pushes back to `release`.

## Normal Release Flow

1. Change a chart under `charts/<chart-name>/`.
2. Bump `version:` in that chart's `Chart.yaml`.
3. Push to the `release` branch.
4. GitHub Actions rebuilds all chart packages and `build/index.yaml`.
5. Consumers use `https://hamropatro.github.io/helm-charts/build` and run `helm repo update` to get the new chart version.

## Validation

Before pushing chart changes, run:

```sh
helm lint charts/<chart-name>
helm package charts/<chart-name> -d /tmp/helm-chart-test
```

For all charts, run:

```sh
mkdir -p build
for chart in charts/*; do
  [ -d "$chart" ] || continue
  helm lint "$chart"
  helm package "$chart" -d build
done
helm repo index build/
```

## Rollback

If a workflow change breaks release, revert the workflow file and rerun the failed GitHub Action.

If a chart package was published with the wrong content, bump the chart version again and publish a corrected package.
