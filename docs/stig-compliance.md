# RKE2 STIG compliance

Mapped against the **Rancher Government Solutions RKE2 Security Technical Implementation Guide, V2R7 (2026-05-13)** - 21 findings, 4 CAT I and 17 CAT II.

Fifteen of the 21 are settings a package definition can carry. All fifteen are satisfied by `templates/distro.yaml.tmpl`. The remaining six are operational or workload concerns that no definition can assert; they are listed with what this repository does toward them.

## Settings this definition enforces

| Finding  | Sev    | Required                                                                                                                                                  | Where in the template                                                     |
| -------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| V-254562 | CAT I  | `anonymous-auth=false`                                                                                                                                    | `kube-apiserver-arg`                                                      |
| V-254561 | CAT I  | `authorization-mode=Webhook`                                                                                                                              | `kubelet-arg`                                                             |
| V-254559 | CAT I  | `read-only-port=0`                                                                                                                                        | `kubelet-arg`                                                             |
| V-254553 | CAT I  | `tls-min-version` >= `VersionTLS12` and the three named ECDSA cipher suites, on the apiserver, controller manager and scheduler      | `kube-apiserver-arg`, `kube-controller-manager-arg`, `kube-scheduler-arg` |
| V-254554 | CAT II | `use-service-account-credentials=true`                                                                                                                    | `kube-controller-manager-arg`                                             |
| V-254555 | CAT II | `audit-policy-file` set, `audit-log-mode=blocking-strict`, `profile: cis`, and an audit policy of secrets at `Metadata` / everything at `RequestResponse` | `audit:`, `kube-apiserver-arg`, `profile`                                 |
| V-254556 | CAT II | `bind-address=127.0.0.1`                                                                                                                                  | `kube-controller-manager-arg`                                             |
| V-254557 | CAT II | kubelet `anonymous-auth=false`                                                                                                                            | `kubelet-arg`                                                             |
| V-254563 | CAT II | `audit-log-maxage=30`                                                                                                                                     | `kube-apiserver-arg`                                                      |
| V-254564 | CAT II | `write-kubeconfig-mode: "0600"`                                                                                                                           | `config.write-kubeconfig-mode`                                            |
| V-254565 | CAT II | unnecessary components disabled                                                                                                                           | `disable`, `disable-kube-proxy`, `disable-cloud-controller`               |
| V-254568 | CAT II | `streaming-connection-idle-timeout=5m`                                                                                                                    | `kubelet-arg`                                                             |
| V-254569 | CAT II | `protect-kernel-defaults=true`                                                                                                                            | `config.protect-kernel-defaults`                                          |
| V-254571 | CAT II | `pod-security-admission-config-file` set, with `enforce`/`audit`/`warn` all `restricted` at `latest`                                                      | `podSecurity:`                                                            |
| V-254572 | CAT II | `authorization-mode=RBAC,Node`                                                                                                                            | `kube-apiserver-arg`                                                      |

Two of these are satisfied indirectly, and it is worth knowing how. cargoship turns `.spec.config.engine.audit` into `/etc/rancher/rke2/audit.yaml` and sets `audit-policy-file` to it, and turns `.spec.config.engine.podSecurity` into `/etc/rancher/rke2/pss.yaml` and sets `pod-security-admission-config-file` to it - adding the `kind` and `apiVersion` of each file itself. So the template supplies policy content, not the flags; the flags appear in the rendered `config.yaml` on the node.

## What was changed to reach this

### V-254555: the audit policy logged nothing

The definition carried:

```yaml
audit:
  rules:
    - level: None
```

`level: None` matches every request and logs none of it, so the cluster satisfied the parts of V-254555 that ask for `audit-policy-file` and `audit-log-mode` while auditing nothing at all. It now carries the policy the STIG's check text requires verbatim.

**This is loud.** `RequestResponse` on `*` logs full request and response bodies for every resource. Size `/var/lib/rancher/rke2/server/logs` accordingly, and expect the audit log to dominate control-plane disk I/O on a busy cluster. That cost is what the requirement buys, and it is not tunable without taking the finding.

### V-254553: the cipher list is now exactly the three the STIG names

The check text asks only that the list *contain* the three suites, so a longer list also passes. This repository carries exactly the three:

```text
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
```

Three `TLS_ECDHE_RSA_*` suites were dropped to get here. That is safe on a default cluster, and worth understanding before you supply your own PKI: a TLS 1.2 cipher suite names the authentication algorithm, so an ECDSA-only list can only serve a certificate with an EC key. RKE2 generates its leaf CAs - `server-ca`, `client-ca`, `request-header-ca` and the etcd pair - as ECDSA P-256, and the serving certificates they sign inherit that, so the three suites above negotiate normally. Only `root-ca`, `intermediate-ca` and `service.key` are RSA, and none of those terminates TLS.

**A custom PKI whose leaf CAs are RSA will fail to serve TLS 1.2 at all** - there would be no mutually acceptable suite, rather than a weaker one. If you adapt `generate-custom-ca-certs.sh`, keep the leaf CAs on `prime256v1`. TLS 1.3 is unaffected either way: its suite names carry no authentication algorithm, and Go does not let `tls-cipher-suites` configure them.

## Findings no definition can assert

| Finding  | Sev    | Why it is not a setting                                   | What this repository does                                                                                   |
| -------- | ------ | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| V-254566 | CAT II | The PPSM CAL ports are site policy, not a cluster default | Nothing. Review against your own CAL                                                                        |
| V-254567 | CAT II | Concerns secrets in workload environment variables        | Nothing. A workload admission policy is the control                                                         |
| V-254570 | CAT II | Namespace isolation of already-running resources          | Pod security admission is enforced `restricted` cluster-wide                                                |
| V-254574 | CAT II | Removing pods still running superseded images             | Nothing. Operational, at upgrade time                                                                       |
| V-254575 | CAT II | Keeping images current                                    | `render.yaml` chases upstream weekly and opens a pull request, which is the mechanism this finding asks for |
| V-268321 | CAT II | Requires images verified from RGS's Carbide registry      | Packages are cosign-signed on publish, but images come from `docker.io/rancher`, **not** Carbide. See below |

### V-268321 is the one gap

The fix text requires images pulled and verified from the Rancher Government Carbide registry. These definitions pull from `docker.io/rancher/*`, which is upstream Rancher rather than Carbide, so a Carbide-based accreditation would take this finding.

Closing it means pointing `imageConfig.images` at Carbide references and carrying Carbide pull credentials. That is a deliberate decision about where images come from, not a template tweak, so it is left open rather than half-done.

## Verifying a cluster

The STIG's own check texts mostly read the running process, so they apply unchanged to a node this package installed:

```sh
/bin/ps -ef | grep kube-apiserver | grep -v grep
cat /etc/rancher/rke2/config.yaml
cat /etc/rancher/rke2/audit.yaml
cat /etc/rancher/rke2/pss.yaml
```

Note the file names: the STIG's fix text names `/etc/rancher/rke2/audit-policy.yaml` and `rke2-pss-custom.yaml` as examples, while cargoship writes `audit.yaml` and `pss.yaml` and points the flags at those. The check text allows this - it reads the path out of the flag rather than assuming one.

### Scope

V-254564 also requires ownership and permissions on files under `/etc/rancher/rke2` and `/var/lib/rancher/rke2`. The definition sets `write-kubeconfig-mode`, which is the part it can; the rest is applied to the host at install time and should be verified on the node.

The namespace exemptions under `podSecurity` (`kube-system`, `rook-ceph`, `zarf`) are not checked by V-254571, which validates only the `defaults` block. Each exemption is nonetheless an accepted risk - a namespace there runs outside `restricted` - so remove any your cluster does not use.
