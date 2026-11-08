# Refactoring this repo onto `cargoship dev render`

Plan only. Nothing here is implemented yet. The model is [Rendering Definitions in CI](https://github.com/colonel-byte/cargoship/blob/main/docs/guides/rendering-in-ci.md).

## Where the repo is today

One hand-written `distro.yaml`, pinned to `1.35.0-rke2r3` in about 35 places: `metadata.version`, `spec.version`, four RPM URLs and their four target paths, and 27 image tags. A version bump is a manual edit across all of them, which is why the repo has sat on `1.35.0-rke2r3` while upstream has reached `1.37.2-rke2r1`.

Five things are wrong independently of the refactor, and each is a one-line fix:

| Where                              | What                                                                                                                                          | Fix                                                          |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `distro.yaml:18`                   | `$schema=.../schema/zarf-v1alpha1-distro-package-schema.json` - that file no longer exists upstream, so the editor header resolves to nothing | `distro-v1alpha1-schema.json`                                |
| `distro.yaml:19-20`                | `apiVersion: zarf.dev/v1alpha1`, `kind: ZarfDistro` - still accepted as a deprecated alias, but not the current names                         | `colonel-byte.dev/v1alpha1`, `kind: Distro`                  |
| `distro.yaml` RPM sources          | no `shasum:` on any of the four. Rancher re-cuts release assets in place, so a changed RPM is pulled silently                                 | add `shasum:` per file (the render step below supplies them) |
| `.github/workflows/release.yml:28` | `setup-cargoship@main` - a branch ref, unpinned, and the guide's whole argument against letting a tool version move on its own                | pin to a SHA + `# v1.0.0`, version from a repo variable      |
| `.github/workflows/release.yml:64` | `--signing-key-pass ${{ secrets.COSIGN_PASS }}` interpolated into a `run:` block                                                              | bind through `env:` like `COSIGN_KEY` already is             |

Two more worth a decision rather than a fix:

- `release.yml:65` verifies against `colonel-byte/zarf-slim-init`'s `cosign.pub` while signing with this repo's `COSIGN_KEY`, and the `cosign.pub` committed here is unused. Either the local key is dead weight or the `--key` URL is wrong.
- `core/cache.json` is **zero bytes** and is shipped to `/var/lib/rancher/rke2/agent/images/.cache.json`. If that is deliberate (a placeholder so the agent does not write its own), it wants a comment; if not, drop the `spec.files` entry.

## Target shape

```text
.
|- templates/
|  |- distro.yaml.tmpl
|  |- values.yaml.tmpl
|  |- values.schema.json.tmpl        # optional, see "Should values exist at all"
|- versions/
|  |- common.yaml                    # everything that does not vary: sysctls, fapolicyd, podSecurity, cilium manifest, CIS args
|  |- 1.35.10-rke2r1.yaml            # version, rpmVersion, images, rpm shasums
|  |- 1.37.2-rke2r1.yaml
|- packages/                         # rendered, never hand-edited
|  |- 1.35.10-rke2r1/{distro.yaml,values.yaml,values.schema.json}
|  |- 1.37.2-rke2r1/
|- hack/new-version.sh               # the only new code in this plan
|- .github/workflows/render.yaml     # new: render -> pull request
|- .github/workflows/release.yml     # reworked: publish what the registry does not have
```

`distro.yaml` at the root goes away. Everything in it moves to `templates/distro.yaml.tmpl` with the version-varying parts replaced by `{{ .Version }}`, `{{ .RPMVersion }}`, `{{ range .Images }}` and `{{ .RPMShasums.<name> }}`.

### The one thing that cannot be templated

`dev render` touches no network and has no `sha256`, `downloadToCache` or `env` helper - deliberately, so a render is reproducible. So the image list and the four RPM digests have to arrive as data, in `versions/<version>.yaml`.

That is what `hack/new-version.sh <upstream-tag>` is for, and it is roughly thirty lines of shell:

1. `gh release download <tag> --repo rancher/rke2 --pattern 'rke2-images-core.linux-amd64.txt' --pattern 'rke2-images-cilium.linux-amd64.txt'` - the image list is an upstream release asset, so nobody transcribes 27 tags again.
2. `curl -fsSL <rpm-url> | sha256sum` for `rke2-common`, `rke2-selinux`, `rke2-server`, `rke2-agent`. This doubles as the guide's "check that what a release publishes is still fetchable before writing its values file" - a 404 means the revision was retired and the script exits without writing anything.
3. Emit `versions/<version>.yaml`.

It runs locally (`hack/new-version.sh v1.37.2+rke2r1`) and as step 2 of the render workflow. Keeping the fetching in a script rather than in the workflow is what makes a new version reproducible by hand.

Two things to decide while writing it:

- **Which image list.** `rke2-images-core` + `rke2-images-cilium` is close to the 27 images listed today. `rke2-images-all` is a superset and includes calico/canal images this package does not want. Recommend core+cilium.
- **`rke2-selinux` version.** Hard-coded `0.22-1.el9` today, and it moves on its own schedule rather than with the k8s minor. Resolve it from the `common/centos/9/noarch/` repodata in the script, or pin it in `versions/common.yaml` and let Dependabot-by-hand move it. Recommend resolving it, since a retired selinux RPM breaks the build the same way a retired server RPM does.

### Acceptance test for the migration

Render `versions/1.35.0-rke2r3.yaml` and diff the result against today's committed `distro.yaml`. Modulo the five fixes above, it should be byte-identical. That is the proof the template did not quietly change the package, and it is worth one commit of its own before any newer version is added.

### Should `values` exist at all

Today the podSecurity exemptions, the disabled charts and the cilium manifest are frozen into the definition: a cluster cannot change them without a rebuild. Cargoship's own rke2 examples expose them through `values.yaml` + `values.schema.json` with `spec.values.mappings`, so one package installs with or without wireguard encryption, with different namespace exemptions, and so on.

This is a genuine feature, not cleanup, and it is also where most of the template work is. **Recommend splitting it out**: migrate to templates first with the values surface left exactly as frozen as it is today, and add `values.yaml` in a second change if a cluster ever asks for it. If one already has, do it now and copy the mappings block from `magefiles/templates/rke2-distro.yaml.tmpl` upstream.

## The render workflow

Straight from the guide, with the upstream chase as step 2:

```yaml
on:
  push:
    branches: [main]
    paths: [templates/**, versions/**, hack/**]
  schedule: [{cron: 41 6 * * 1}]
  workflow_dispatch:
permissions: {contents: read}
concurrency: {group: render-definitions, cancel-in-progress: false}
```

One job, `contents: write` + `pull-requests: write` on the job, every `uses:` pinned to a SHA, `BRANCH: chore/render-definitions` force-pushed so successive weekly runs collapse into one open pull request. Steps:

1. Checkout, then `colonel-byte/setup-cargoship` pinned to a SHA with `version: ${{ vars.CARGOSHIP_VERSION }}`.
2. **Chase upstream**: for each minor line tracked in a small `versions/tracked.yaml` (`1.35`, `1.36`, `1.37`), resolve the newest acceptable `rke2rN` tag with `gh release list --repo rancher/rke2` and run `hack/new-version.sh` if it moved. Skip rather than fail on a retired revision.
3. **Render**: loop `versions/*.yaml`, skipping `common.yaml` and `tracked.yaml`, `rm -rf packages/$name` then `cargoship dev render -t templates/distro.yaml.tmpl -f versions/common.yaml -f "$input" -o "packages/$name"`. No `--no-validate`.
4. `git add --all -- packages versions`, bail with `changed=false` if nothing moved, otherwise force-push the branch and `gh pr create`/`gh pr edit`.

Put the upstream tag diff in the pull request body, the way `refresh-examples.yaml` surfaces its pin diff - the rendered churn is hundreds of lines all saying the same thing, and the version bump is the reviewable part.

**No `cargoship create` in this workflow.** A build fetches every image the definition declares, which is gigabytes and the thing a hosted runner's disk runs out of.

The pull request will have no checks on it, because one opened with `GITHUB_TOKEN` does not start this repository's own `pull_request` workflows. That is fine here - there are none. If a validate check is ever added, switch to a GitHub App token.

## Publishing when a new version lands

You asked about git tags. My recommendation is **the registry as the source of truth, with tags as a record rather than as the trigger**, which is what cargoship does to its own examples in `publish-example.yaml`.

The problem with tag-as-trigger is that this repo holds N versions at once. A tag is repo-wide, so a tag push cannot say *which* package moved without diffing, and a run that re-renders three lines at once needs three tags. You end up maintaining tag bookkeeping to express something the registry already knows.

### Recommended: publish what the registry does not have

`release.yml` runs on push to `main`, and before building anything it asks GHCR:

```sh
for dir in packages/*/; do
  name=$(yq -r '.metadata.name'    "$dir/distro.yaml")   # rancher-rke2
  version=$(yq -r '.metadata.version' "$dir/distro.yaml")
  # publish only if <name>:<version> is not already a tag on the package
done
```

`cargoship publish` derives its reference from the definition - `<registry>/<metadata.name>:<metadata.version>` - so the probe and the push agree by construction, with nothing naming a version in the workflow.

What this buys:

- **Idempotent.** A push to `main` that changed only the README publishes nothing. Re-running a failed job republishes only what did not land.
- **Self-healing.** A version whose push timed out is picked up by the next run, including the weekly render's run.
- **Adding a version is adding a directory.** No workflow edit, no tag to remember.
- **Reviewed.** The render pull request is the human gate, which is where it belongs - the diff a reviewer should read is the definition, not a tag name.

Then, after a successful push, the job writes an annotated git tag as the record: `rancher-rke2/1.37.2-rke2r1`. Path-prefixed so multiple lines coexist, created from the publish rather than driving it. That gives the immutable git coordinate for "this is what we shipped" without the trigger problem.

Also worth it: a floating `--tag latest` push for the highest version, so consumers can pull `oci://ghcr.io/colonel-byte/cargoship/rancher-rke2:latest`. The second push of the same content is cheap - the registry already holds the layers.

### If you want tag-as-trigger anyway

It is a defensible design, just a manual one. Make the tag name *be* the package directory:

```yaml
on:
  push:
    tags: ['packages/*']
```

`git tag packages/1.37.2-rke2r1 && git push --tags` publishes exactly `packages/1.37.2-rke2r1/`, and the trigger carries the selection with no probing and no `yq`. You get a per-publish human gate and a trivially readable audit trail; you give up the automation you asked for, since somebody has to push the tag every time a render merges. If the gate matters more than the automation, take this.

What I would not do is trigger on a version tag with no path prefix (`v1.37.2-rke2r1`), because it reads like a tag describing *this repository's* version and this repository has no version of its own.

## Suggested order

1. The five one-line fixes to the current `distro.yaml` and `release.yml`, and a decision on `cosign.pub` and the empty `core/cache.json`. Independent of everything below, and the signing-pass one is a secret in a shell argument.
2. `templates/` + `versions/common.yaml` + `versions/1.35.0-rke2r3.yaml`, and the byte-identical render diff as the acceptance test. Delete the root `distro.yaml`.
3. `hack/new-version.sh`, with `versions/1.37.2-rke2r1.yaml` as its first output.
4. `.github/workflows/render.yaml`.
5. Rework `release.yml` to the registry probe, plus the record tag.
6. `values.yaml` + `values.schema.json`, only if a cluster needs to override something.

Each step is reviewable on its own and leaves the repo working.
