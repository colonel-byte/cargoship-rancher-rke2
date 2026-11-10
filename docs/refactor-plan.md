# Refactoring this repo onto `cargoship dev render`

Plan only - nothing here is implemented. The model is [Rendering Definitions in CI](https://github.com/colonel-byte/cargoship/blob/main/docs/guides/rendering-in-ci.md).

The root `distro.yaml` is deleted by step 2. That is agreed, not a question left open.

## What was verified before planning

Three things that decide the shape of this refactor, all checked against upstream rather than assumed:

| Claim                                                                      | Result                                                                                                                                                                       |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The 26 images in `distro.yaml` can be derived from upstream release assets | **Yes, exactly.** `cat rke2-images-core.linux-amd64.txt rke2-images-cilium.linux-amd64.txt` for `v1.35.0+rke2r3` is byte-for-byte the committed list, in the committed order |
| The RPM URLs are mechanical                                                | **Yes.** `rpm.rancher.io/rke2/latest/<minor>/centos/9/x86_64/rke2-<pkg>-<rpmVersion>-0.el9.x86_64.rpm` returns 200 for 1.35.0~rke2r3 and 1.37.2~rke2r1 alike                 |
| `rke2-selinux` is pinned correctly                                         | **No.** Pinned at `0.22-1`; `0.24-1` is current. The `common/centos/9/noarch` repodata lists every available version, so it resolves in three lines                          |

The consequence of the first row is that the image list needs no curation: the script writes upstream's two files through verbatim, and the acceptance test in step 2 should come out clean.

### One blocker

Local `cargoship` is **0.37.0**, which has no `dev` command at all. `dev render` is in **v0.39.0** (the current release). Upgrade the local binary before step 2, and set the repository variable `CARGOSHIP_VERSION` to `v0.39.0` for CI.

## Where the repo is today

One hand-written `distro.yaml` pinned to `1.35.0-rke2r3` in about 35 places: `metadata.version`, `spec.version`, four RPM URLs and their four target paths, and 26 image tags. A version bump is a manual edit across all of them, which is why the repo sits three minor lines behind upstream.

Defects found on the way through, each a one-line fix and all independent of the refactor:

| Where                              | What                                                                                                            | Fix                                             |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| `distro.yaml:18`                   | `$schema` pointed at `zarf-v1alpha1-distro-package-schema.json`, which no longer exists upstream                | **already fixed in the working tree**           |
| `distro.yaml:19-20`                | `apiVersion: zarf.dev/v1alpha1`, `kind: ZarfDistro` - accepted as deprecated aliases, but not the current names | `colonel-byte.dev/v1alpha1`, `kind: Distro`     |
| `distro.yaml` RPM sources          | no `shasum:` on any of the four. Rancher re-cuts release assets in place, so a changed RPM is pulled silently   | add `shasum:` per file; step 3 supplies them    |
| `distro.yaml` selinux pin          | `0.22-1.el9` while `0.24-1` is current                                                                          | resolve from repodata in step 3                 |
| `.github/workflows/release.yml:28` | `setup-cargoship@main` - a branch ref, unpinned                                                                 | pin to a SHA with a `# v1.0.0` comment          |
| `.github/workflows/release.yml:64` | `--signing-key-pass ${{ secrets.COSIGN_PASS }}` interpolated into a `run:` block                                | bind through `env:`, as `COSIGN_KEY` already is |

Two that need a decision rather than a fix:

- `release.yml:65` verifies against `colonel-byte/zarf-slim-init`'s `cosign.pub` while signing with this repo's `COSIGN_KEY`, and the `cosign.pub` committed here is never read. Either the local key is dead weight or the `--key` URL is wrong.
- `core/cache.json` is **zero bytes** and ships to `/var/lib/rancher/rke2/agent/images/.cache.json`. If that is a deliberate placeholder it wants a comment; if not, drop the `spec.files` entry.

## Target shape

```text
.
|- templates/
|  |- distro.yaml.tmpl               # everything invariant stays literal in here
|- versions/
|  |- tracked.yaml                   # the minor lines to chase: 1.35, 1.36, 1.37
|  |- 1.35.0-rke2r3.yaml             # version, rpmVersion, minor, selinux, rpm shasums, images
|  |- 1.37.2-rke2r1.yaml
|- packages/                         # rendered, never hand-edited
|  |- 1.35.0-rke2r3/distro.yaml
|  |- 1.37.2-rke2r1/distro.yaml
|- hack/new-version.sh               # the only new code in this plan
|- .github/workflows/render.yaml     # new: chase upstream, render, open a pull request
|- .github/workflows/release.yml     # reworked: publish what the registry does not have
```

### No `common.yaml`

The guide's layout has one, and this repo does not need it. The sysctls, the fapolicyd rules, the podSecurity configuration, the CIS kube-apiserver arguments and the cilium `HelmChartConfig` do not vary by version - so they stay as **literal YAML inside the template**, where they read exactly as they do today. Routing them through a values file would buy nothing and cost a `{{ .sysctl | toYaml | indent 8 }}` at every one.

Only what varies by version becomes a value, which makes each version file about 35 lines:

```yaml
# versions/1.35.0-rke2r3.yaml - generated by hack/new-version.sh, do not hand-edit
version: 1.35.0-rke2r3
rpmVersion: 1.35.0~rke2r3
minor: "1.35"
selinux:
  version: 0.24-1
  shasum: <sha256>
rpms:
  common: <sha256>
  server: <sha256>
  agent: <sha256>
images:
  - docker.io/rancher/rke2-runtime:v1.35.0-rke2r3
  # ... 25 more, verbatim from upstream's two asset files
```

One gotcha to confirm on the first render: `dev render` merges values files into a plain map, so a template reads the keys **exactly as the file spells them** - `{{ .version }}`, not `{{ .Version }}`. The capitalised form in cargoship's own `magefiles/templates/` is a Go struct field and does not apply here.

### The one thing that cannot be templated

`dev render` touches no network and has no `sha256`, `downloadToCache` or `env` helper, deliberately, so that a render is reproducible. So the image list and the RPM digests have to arrive as data. That is the whole job of `hack/new-version.sh <upstream-tag>`, which is about thirty lines:

1. Derive the four string forms from the tag - this is all of the version handling:

| Form         | From `v1.35.0+rke2r3` | Used in                                        |
| ------------ | --------------------- | ---------------------------------------------- |
| `version`    | `1.35.0-rke2r3`       | `metadata.version`, `spec.version`, image tags |
| `rpmVersion` | `1.35.0~rke2r3`       | RPM file names                                 |
| `minor`      | `1.35`                | the RPM repository path                        |
| the tag      | `v1.35.0+rke2r3`      | the `gh release` call                          |

2. `gh release download <tag> --repo rancher/rke2 --pattern 'rke2-images-core.linux-amd64.txt' --pattern 'rke2-images-cilium.linux-amd64.txt'`, concatenated in that order into `images`. Verified byte-identical to what is committed today, so nobody transcribes 26 tags again.
3. Resolve the newest `rke2-selinux` from `common/centos/9/noarch/repodata`, which is `repomd.xml` -> the `primary.xml.gz` href -> `<location href>` filtered to `rke2-selinux` -> `sort -V | tail -1`.
4. `curl -fsSL <url> | sha256sum` for `rke2-common`, `rke2-server`, `rke2-agent` and the selinux RPM. This doubles as the guide's "check that what a release publishes is still fetchable before writing its values file" - a 404 means the revision was retired, and the script exits **without writing anything** rather than producing a definition that renders and cannot be installed.
5. Write `versions/<version>.yaml`.

It runs by hand (`hack/new-version.sh v1.37.2+rke2r1`) and from the render workflow. Keeping the fetching in a script rather than inline in the workflow is what makes a new version reproducible locally. It needs one assert-based self-check: four non-empty shasums and a non-empty image list before it writes.

## Step 2's acceptance test

Run `hack/new-version.sh v1.35.0+rke2r3`, render it, and diff `packages/1.35.0-rke2r3/distro.yaml` against the `distro.yaml` being deleted. Modulo the known fixes - `apiVersion`/`kind`, the added `shasum:` lines, and the selinux bump - it should be identical. That is the proof the template did not quietly change the package, and it is worth its own commit before any newer version is added.

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

One job, every `uses:` pinned to a SHA, and `BRANCH: chore/render-definitions` force-pushed so successive weekly runs collapse into a single open pull request. The job needs **no** `contents: write` or `pull-requests: write`: the pull request is pushed with a GitHub App token, which carries both, and the workflow's own token never needs more than reading the repository. Steps:

1. Mint the app token, then checkout **with it**, then `colonel-byte/setup-cargoship` pinned to a SHA with `version: ${{ vars.CARGOSHIP_VERSION }}`. This repository has the same secrets as cargoship itself, so it is the same app `refresh-examples.yaml` uses, under the same secret names:

   ```yaml
   - name: Get app token
     id: app-token
     uses: actions/create-github-app-token@64ab729a6294000f5aa4a4bc0b73d1f32bc45510 # v3.3.0
     with:
       client-id: ${{ secrets.RELEASE_PLEASE_GITHUB_APP_ID }}
       private-key: ${{ secrets.RELEASE_PLEASE_GITHUB_APP_SECRET }}

   - name: Checkout
     uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
     with:
       token: ${{ steps.app-token.outputs.token }}
   ```

   The checkout has to take the token too, not just the `gh pr` step - it is what leaves the pushing remote authenticated as the app, so the force-push in step 4 counts as the app rather than as the workflow.
2. For each minor line in `versions/tracked.yaml`, resolve the newest `rke2rN` tag with `gh release list --repo rancher/rke2` and run `hack/new-version.sh` if it moved. A retired revision skips rather than fails the job.
3. Render every version file: `rm -rf "packages/$name"` then `cargoship dev render -t templates/distro.yaml.tmpl -f "versions/$name.yaml" -o "packages/$name"`. The `rm -rf` matters - a template that stops writing a file should stop carrying it. No `--no-validate`: the schema check is the only free check this workflow gets.
4. `git add --all -- packages versions`, exit with `changed=false` if nothing moved, otherwise force-push the branch and `gh pr create` / `gh pr edit`.

Put the resolved-tag diff in the pull request body, the way cargoship's `refresh-examples.yaml` surfaces its pin diff. The rendered churn is hundreds of lines all saying the same thing; the version bump is the reviewable part.

**No `cargoship create` in this workflow.** A build fetches every image the definition declares - gigabytes, and the thing a hosted runner's disk runs out of. Building stays on the release path.

Because the pull request is opened by the app rather than by `GITHUB_TOKEN`, it **does** start this repository's own `pull_request` workflows - that is the whole reason for the app token, and the reason `refresh-examples.yaml` uses one. There are no such workflows here yet, so nothing changes today; the value is that adding a validate check later needs no rework, and a branch protection rule requiring a check can actually be satisfied.

Pass `${{ steps.app-token.outputs.token }}` as `GH_TOKEN` to the `gh pr` step. A personal access token would also work and is the thing to avoid: it ties the repository's automation to one person's account and expires on their schedule.

## Publishing when a new version lands

You preferred git tags. My recommendation is **the registry as the source of truth, with a tag as the record rather than the trigger** - what cargoship does to its own examples in `publish-example.yaml`.

The problem with tag-as-trigger is that this repo will hold several versions at once. A tag is repo-wide, so a tag push cannot say *which* package moved without diffing, and a render that bumps three lines needs three tags. You end up maintaining bookkeeping to express something the registry already knows.

### Recommended

`release.yml` runs on push to `main` and, before building anything, asks GHCR which references already exist:

```sh
for dir in packages/*/; do
  name=$(yq -r '.metadata.name'    "$dir/distro.yaml")   # rancher-rke2
  version=$(yq -r '.metadata.version' "$dir/distro.yaml")
  # build and publish only if <name>:<version> is not already a tag on the package
done
```

`cargoship publish` derives its reference from the definition - `<registry>/<metadata.name>:<metadata.version>`, see `pkg/coci/utils.go` - so the probe and the push agree by construction, with nothing in the workflow naming a version.

What it buys:

- **Idempotent.** A push to `main` touching only the README publishes nothing. Re-running a failed job republishes only what did not land.
- **Self-healing.** A version whose push timed out is picked up by the next run, including the weekly render's.
- **Adding a version is adding a directory.** No workflow edit, no tag to remember.
- **Still reviewed.** The render pull request is the human gate, and the diff a reviewer should read is the definition, not a tag name.

Then, after a successful push, the job writes an annotated tag as the record: `rancher-rke2/1.37.2-rke2r1`. Path-prefixed so several lines coexist, created *from* the publish rather than driving it. That keeps an immutable git coordinate for what shipped without the trigger problem.

Worth adding: a floating `--tag latest` push for the highest version, so consumers can pull `oci://ghcr.io/colonel-byte/cargoship/rancher-rke2:latest`. The second push of the same content is cheap, since the registry already holds the layers.

### If you want tag-as-trigger anyway

Defensible, just manual. Make the tag name *be* the package directory:

```yaml
on:
  push:
    tags: ['packages/*']
```

`git tag packages/1.37.2-rke2r1 && git push --tags` then publishes exactly `packages/1.37.2-rke2r1/`, with no probing and no `yq` - the trigger carries the selection. You get a per-publish gate and a readable audit trail, and you give up the automation, since somebody pushes a tag every time a render merges. If the gate matters more than the automation, take this.

What I would avoid is a bare version tag (`v1.37.2-rke2r1`), which reads as *this repository's* version, and this repository has none of its own.

## Order

| # | Change                                                                                                                      | Leaves the repo                       |
| - | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| 1 | The one-line fixes to `distro.yaml` and `release.yml`; decide on `cosign.pub` and the empty `core/cache.json`               | working, publishing as it does now    |
| 2 | `templates/distro.yaml.tmpl` + `versions/1.35.0-rke2r3.yaml`, the byte-identical render diff, delete the root `distro.yaml` | working, one package, rendered        |
| 3 | `hack/new-version.sh`, with `versions/1.37.2-rke2r1.yaml` as its first output                                               | working, two packages                 |
| 4 | `.github/workflows/render.yaml`                                                                                             | chasing upstream weekly               |
| 5 | Rework `release.yml` to the registry probe, plus the record tag                                                             | publishing new versions automatically |
| 6 | `values.yaml` + `values.schema.json`, only if a cluster needs to override something                                         | overridable at install time           |

Step 1 is worth doing first regardless, because the signing-pass fix puts a secret in a shell argument today.

Step 6 is a genuine feature rather than cleanup, and it is where most of the remaining template work is. Today the podSecurity exemptions, the disabled charts and the cilium manifest are frozen into the definition: a cluster cannot change them without a rebuild. Cargoship's own rke2 examples expose them through `values.yaml` + `values.schema.json` and `spec.values.mappings`, so one package installs with or without wireguard encryption and with different namespace exemptions. **Defer it** unless a cluster has already asked - and if one has, copy the mappings block from `magefiles/templates/rke2-distro.yaml.tmpl` upstream rather than writing it.
