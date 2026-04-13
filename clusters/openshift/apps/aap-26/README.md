# AAP 2.6 GitOps layout

This folder deploys Ansible Automation Platform 2.6 on OpenShift through Argo CD.

**Start here for the full template guide:** [repository root `README.md`](../../../../README.md) (how to fork, customize, and apply `aap-project.yaml` / `aap26-application.yaml`).

## Structure

- `base/`: shared resources (namespace, OperatorGroup, Subscription, AAP CR)
- `overlays/dev`, `overlays/stage`, `overlays/prod`: environment patches
- `secrets/`: reference examples (Sealed Secrets; Vault+ESO copies for prod also live under `overlays/prod/` for Argo Kustomize path rules)

## Secret patterns

Choose one approach and keep plaintext secrets out of Git.

- **Sealed Secrets:** `secrets/sealedsecret-aap-admin-password-prod.yaml`
- **HashiCorp Vault + ESO (used by prod sync):**
  - `overlays/prod/secretstore-vault.yaml`
  - `overlays/prod/externalsecret-aap-admin-password-prod.yaml`

## Notes

- Root `kustomization.yaml` in this folder points to `overlays/prod`.
- Set Argo CD `Application.spec.source.path` to `clusters/openshift/apps/aap-26` for prod, or to `clusters/openshift/apps/aap-26/overlays/<env>` for dev/stage.
- Verify AAP CR fields against your installed operator version.
- For Vault, update URL, KV path, auth mount, and role in `overlays/prod/secretstore-vault.yaml`.
