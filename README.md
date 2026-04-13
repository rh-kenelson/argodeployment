# Argo CD template: Ansible Automation Platform 2.6 on OpenShift

This repository is a GitOps template for deploying **Red Hat Ansible Automation Platform 2.6** on **OpenShift** with **Argo CD** (for example OpenShift GitOps in `openshift-gitops`).

## What is in this repo

| Path | Purpose |
|------|---------|
| `aap-project.yaml` | Argo CD `AppProject` (`aap`) — scopes allowed repos and destination namespace |
| `aap26-application.yaml` | Argo CD `Application` that points at the Kustomize path below |
| `clusters/openshift/apps/aap-26/` | Kustomize layout: namespace, OperatorGroup, Subscription (Red Hat operator), AAP instance, optional Vault+ESO for prod |

Detailed layout and secret options are documented in [`clusters/openshift/apps/aap-26/README.md`](clusters/openshift/apps/aap-26/README.md).

## Prerequisites

- OpenShift cluster with **Red Hat Operator Hub** / `redhat-operators` catalog (entitlements as required).
- **OpenShift GitOps** (or Argo CD) installed; default manifests assume Argo lives in namespace `openshift-gitops`.
- This repo (or your fork) pushed to a Git server Argo CD can reach.
- For External Secrets + Vault: ESO installed, Vault reachable from the cluster, and Kubernetes auth configured for the role you reference in the `SecretStore`.

## How to use this template

### 1. Fork or copy the repo

Use your own Git remote. Replace every placeholder URL with yours.

### 2. Customize manifests

- **`aap-project.yaml`** — set `spec.sourceRepos` to your repo URL(s).
- **`aap26-application.yaml`** — set `spec.source.repoURL`, `targetRevision`, and optionally `spec.source.path`:
  - Default path `clusters/openshift/apps/aap-26` resolves through root `kustomization.yaml` to **`overlays/prod`**.
  - For another environment, point `path` at `clusters/openshift/apps/aap-26/overlays/dev` or `.../overlays/stage`.
- **`clusters/openshift/apps/aap-26/overlays/prod/`** — Vault-backed secrets live here (`secretstore-vault.yaml`, `externalsecret-aap-admin-password-prod.yaml`) so Argo’s Kustomize build stays within each overlay directory. Adjust Vault URL, mount path, role, and remote key to match your environment.
- **`clusters/openshift/apps/aap-26/base/subscription.yaml`** — confirm `channel` and operator `name` match your catalog.

### 3. Deploy the Argo CD objects

From a machine with `oc` and cluster admin (or sufficient rights to create `AppProject` / `Application` in `openshift-gitops`):

```bash
oc apply -f aap-project.yaml
oc apply -f aap26-application.yaml
```

Argo CD will sync from Git and apply Namespace, OperatorGroup, Subscription, then (after the operator installs the CRD) the `AnsibleAutomationPlatform` custom resource. Sync waves and `SkipDryRunOnMissingResource` on the AAP CR reduce ordering issues while the operator installs.

### 4. Verify

```bash
oc get applications.argoproj.io -n openshift-gitops
oc get csv -n aap
oc get ansibleautomationplatform -n aap
```

## Important constraints

- **Kustomize in Argo CD** does not allow `../` references that leave the current kustomization root. Base resources live under `base/`; prod-specific Vault/ESO manifests live under `overlays/prod/`.
- **Secrets** must not be committed in plaintext. Use Vault+ESO (as in prod overlay), Sealed Secrets, or another approved pattern.
- **`AppProject` RBAC** in `aap-project.yaml` is permissive for a quick start; tighten `clusterResourceWhitelist` / `namespaceResourceWhitelist` for production governance.

## Further reading

- App-specific structure and notes: [`clusters/openshift/apps/aap-26/README.md`](clusters/openshift/apps/aap-26/README.md)
