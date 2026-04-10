# AAP 2.6 GitOps Layout

This folder deploys Ansible Automation Platform 2.6 on OpenShift through Argo CD.

## Structure

- `base/`: shared resources
- `overlays/dev`: development patch set
- `overlays/stage`: staging patch set
- `overlays/prod`: production patch set
- `secrets/`: examples for secret delivery patterns

## Secret patterns

Choose one approach and keep plaintext secrets out of Git.

- Sealed Secrets: `secrets/sealedsecret-aap-admin-password-prod.yaml`
- External Secrets: `secrets/externalsecret-aap-admin-password-prod.yaml`

## Notes

- Current root `kustomization.yaml` points to `overlays/prod`.
- Set Argo CD `Application.spec.source.path` to this folder: `clusters/openshift/apps/aap-26`.
- Verify AAP CR fields against your installed operator version.
