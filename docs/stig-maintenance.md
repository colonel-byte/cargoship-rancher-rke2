# Keeping the STIG mapping current

[docs/stig-compliance.md](stig-compliance.md) is a claim about a specific release of a specific document: **Rancher Government Solutions RKE2 STIG, V2R7, 2026-05-13**. DISA re-releases it a few times a year, and a release can add findings, retire them, or change the required value of one that already passes. Nothing in this repository notices that - the render workflow chases *rke2*, not the STIG.

So this is a manual review, and the point of this page is to make it a short one.

## When to do it

- **DISA publishes a new release.** The version is shown at the top of [the STIG's index page](https://www.stigviewer.com/stigs/rancher_government_solutions_rke2). Quarterly is a reasonable cadence to look.
- **Before an accreditation submission**, regardless of when the last review was.
- **When a tracked rke2 minor line is added**, because V-254555 ties the acceptable `profile` value to the rke2 version - `cis-1.6` for v1.24 and older, `cis-1.23` for v1.25+, and the generic `cis` for the October 2023 releases onward. Every line this repository tracks is far past that boundary, so `profile: cis` is correct for all of them, but a new line is the moment to re-read that rule rather than assume.

## Getting the document

The authoritative source is the XCCDF zip from [DISA's STIG library](https://public.cyber.mil/stigs/downloads/). `stigviewer.com` renders the same content but serves its pages from JavaScript, so `curl` gets an empty shell and its JSON and XML links are not fetchable.

What worked for the V2R7 review was a third-party CSV export carrying every finding with its full check and fix text, which is the shape you want - one file, greppable, no per-finding page loads:

```sh
curl -fsSL -o stig.csv \
  "https://www.ignyteplatform.com/stigs/releases/rancher-government-solutions-rke2-security-technical-implementation-guide-v2r7/download/csv/"
```

Substitute the release in the URL (`v2r7` -> `v2r8`). It is a convenience, not an authority: **confirm the version, date and finding count against DISA or stigviewer before relying on it**, and read anything surprising against the official XCCDF.

Listing the findings, which is the first thing to do with it:

```sh
python3 - <<'EOF'
import csv, re, textwrap
rows = list(csv.DictReader(open('stig.csv')))
print(f"{len(rows)} findings\n")
for r in rows:
    print(f"\n##### {r['Finding ID']} [{r['Severity']}] {r['Title'][:80]}")
    for field in ('Check Text', 'Fix Text'):
        body = re.sub(r'\s+', ' ', r[field]).strip()
        print(textwrap.fill(body, 145, initial_indent=f'  {field[:5].upper()}: ',
                            subsequent_indent='         '))
EOF
```

## The review

Work from the **check** text, not the fix text. The fix text says what to type; the check text says what an assessor will accept, and the two differ in ways that matter. V-254553 is the example: its fix text lists three cipher suites, while its check text only requires that the configured list *contain* them. Reading the fix text alone would have made a longer list look like a finding.

1. **Diff the finding list against the table in `stig-compliance.md`.** New IDs and retired IDs are the cheap part.
2. **For each finding already in the table, re-read its check text** and confirm the required value has not moved. A release that changes `audit-log-maxage=30` to `=90`, or adds a cipher suite, does not change the finding ID.
3. **Decide where each new finding lands**: a setting the definition can carry, or something operational. The split in `stig-compliance.md` is the structure to extend.
4. **Change the template, never a rendered file.** Then re-render and validate:

   ```sh
   for input in versions/1.*.yaml; do
     name=$(basename "$input" .yaml)
     rm -rf "packages/$name"
     cargoship dev render -t templates/distro.yaml.tmpl -f "$input" -o "packages/$name"
   done
   cargoship validate packages/*/distro.yaml
   ```

5. **Update `stig-compliance.md`**, including the version and date in its first line. A mapping that does not say which release it was built against is not evidence of anything.

## Two things that are easy to get wrong

**A flag the definition does not mention may still be set.** cargoship turns `.spec.config.engine.audit` into `/etc/rancher/rke2/audit.yaml` and sets `audit-policy-file` to it, and does the same for `.spec.config.engine.podSecurity` -> `pss.yaml` and `pod-security-admission-config-file`, adding each file's `kind` and `apiVersion` itself. So grepping the template for a flag name is not how you check whether a rule passes; read the rendered node config, or read `types/distrocfg/rancher_common.go` in cargoship. Both V-254555 and V-254571 pass this way.

**A setting can be present and inert.** The definition satisfied V-254555's `audit-policy-file` and `audit-log-mode` requirements for as long as this repository existed while its audit policy was `level: None`, which logs nothing. The check text covers the policy *content* for exactly this reason. When a rule names a file, check what is in the file.

## What this review cannot tell you

Everything here is definition-level. A definition cannot assert the host file permissions in V-254564, the PPSM CAL ports in V-254566, or anything about running workloads, and a clean review here is not a clean scan on a node. Run the STIG's own check commands against a real cluster before signing anything:

```sh
/bin/ps -ef | grep kube-apiserver | grep -v grep
cat /etc/rancher/rke2/config.yaml /etc/rancher/rke2/audit.yaml /etc/rancher/rke2/pss.yaml
```
