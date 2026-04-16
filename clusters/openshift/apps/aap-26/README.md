# AAP 2.6 GitOps layout

This folder deploys Ansible Automation Platform 2.6 on OpenShift through Argo CD.

**Start here for the full template guide:** [repository root `README.md`](../../../../README.md) (how to fork, customize, and apply `aap-project.yaml` / `aap26-application.yaml`).

## Structure

- `base/`: shared resources (namespace, OperatorGroup, Subscription, AAP CR)
- `overlays/dev`, `overlays/stage`, `overlays/prod`: environment patches
- `overlays/minimal`: Platform Gateway + Automation Controller only (Hub and EDA disabled)
- `secrets/`: reference examples (Sealed Secrets; Vault+ESO copies for prod also live under `overlays/prod/` for Argo Kustomize path rules)

## Overlays

| Overlay | Components | Secret pattern |
|---------|-----------|---------------|
| `dev` | All (Gateway, Controller, Hub, EDA) | Manual / out-of-band secret |
| `stage` | All | Manual / out-of-band secret |
| `prod` | All | HashiCorp Vault + ESO |
| `minimal` | Platform Gateway + Automation Controller only | Sealed Secrets (LDAP bind password + RH manifest) |

To use the minimal overlay point `Application.spec.source.path` at `clusters/openshift/apps/aap-26/overlays/minimal`.

### Minimal overlay — Red Hat license and LDAP at install time

The minimal overlay wires in both the subscription manifest (RH license) and LDAP configuration so the controller is ready to use immediately after the operator installs it. The patch drives three controller sub-fields:

| Field | Purpose |
|-------|---------|
| `controller.subscription_manifest` | Name of the secret holding `manifest.zip` (activates the RH license) |
| `controller.ldap_cacert_secret` | Name of the secret holding the LDAP CA cert (`ldap-ca.crt`) — required for `ldaps://` or StartTLS |
| `controller.extra_settings` | List of non-sensitive Django LDAP settings written inline in the CR spec |
| `controller.extra_settings_secret` | Name of a secret whose keys are merged into Django settings at runtime — keeps the LDAP bind password out of Git |

**Step 1 — Create the manifest secret** (out-of-band, once per cluster):

```bash
# Download manifest.zip from access.redhat.com → Subscriptions → Manage
oc create secret generic aap-manifest \
  --from-file=manifest.zip=/path/to/manifest.zip \
  -n aap
```

**Step 2 — Create the LDAP CA cert secret** (skip if using plain `ldap://`):

```bash
oc create secret generic aap-ldap-cacert \
  --from-file=ldap-ca.crt=/path/to/ldap-ca.crt \
  -n aap
```

**Step 3 — Seal the LDAP bind password**:

Fill in the encrypted value in `secrets/sealedsecret-aap-ldap-settings-minimal.yaml`:

```bash
echo -n '<ldap-bind-password>' \
  | kubeseal --raw --from-file=/dev/stdin \
      --namespace aap \
      --name aap-ldap-settings-minimal
```

Paste the output into the `AUTH_LDAP_BIND_PASSWORD` field of the template, then uncomment the resource line in `overlays/minimal/kustomization.yaml`.

**Step 4 — Adjust the LDAP settings** in `overlays/minimal/patch-aap-instance.yaml`:

- `AUTH_LDAP_SERVER_URI` — your LDAP/LDAPS endpoint
- `AUTH_LDAP_BIND_DN` — service account DN
- `AUTH_LDAP_USER_SEARCH` — base DN and filter (`uid=` for OpenLDAP, `sAMAccountName=` for AD)
- `AUTH_LDAP_REQUIRE_GROUP` — restrict login to this group (remove field to allow all LDAP users)
- `AUTH_LDAP_USER_FLAGS_BY_GROUP` — map a group DN to AAP superuser

## Custom Resources (CRs) you can declare

The AAP operator ships several CRDs. You can declare them inside an overlay or add them to `base/` if they are shared across all environments.

### `AnsibleAutomationPlatform` (primary CR)

Orchestrates every sub-component through a single resource. This is the CR used in this repo.

```yaml
apiVersion: aap.ansible.com/v1alpha1
kind: AnsibleAutomationPlatform
metadata:
  name: aap
  namespace: aap
spec:
  admin_password_secret: <secret-name>
  image_pull_policy: IfNotPresent
  no_log: false

  # Optional per-component overrides — omit a block to accept operator defaults.
  # Set disabled: true on any component you do not want deployed.
  gateway: {}
  controller: {}
  hub:
    disabled: true   # disable Automation Hub
  eda:
    disabled: true   # disable Event-Driven Ansible
```

Key top-level spec fields:

| Field | Type | Description |
|-------|------|-------------|
| `admin_password_secret` | string | Name of the `Secret` containing the `password` key for the AAP admin user |
| `image_pull_policy` | string | `Always`, `IfNotPresent`, or `Never` |
| `no_log` | bool | Suppress sensitive output in operator logs (set `true` in prod) |
| `gateway` | object | Platform Gateway sub-component config (replicas, resource requests, etc.) |
| `controller` | object | Automation Controller sub-component config |
| `hub` | object | Automation Hub sub-component config; supports `disabled: true` |
| `eda` | object | Event-Driven Ansible sub-component config; supports `disabled: true` |

### `AutomationController` (standalone CR)

Deploy Automation Controller independently — useful when you are not using the unified `AnsibleAutomationPlatform` CR.

```yaml
apiVersion: automationcontroller.ansible.com/v1beta1
kind: AutomationController
metadata:
  name: controller
  namespace: aap
spec:
  admin_password_secret: <secret-name>
  replicas: 1
```

### `AutomationHub` (standalone CR)

Deploy Automation Hub independently.

```yaml
apiVersion: hub.ansible.com/v1beta1
kind: AutomationHub
metadata:
  name: hub
  namespace: aap
spec:
  admin_password_secret: <secret-name>
```

### `EDA` (standalone CR)

Deploy Event-Driven Ansible independently.

```yaml
apiVersion: eda.ansible.com/v1alpha1
kind: EDA
metadata:
  name: eda
  namespace: aap
spec:
  automation_server_url: https://<controller-route>
```

### `AutomationGateway` (standalone CR)

Deploy the Platform Gateway independently.

```yaml
apiVersion: gateway.ansible.com/v1alpha1
kind: AutomationGateway
metadata:
  name: gateway
  namespace: aap
spec: {}
```

> **Note:** Always verify field names against the CRD shipped with your installed operator version — run `oc explain AnsibleAutomationPlatform.spec` on the cluster to see the live schema.

## Where to place a Subscription manifest

The operator `Subscription` is declared in `base/subscription.yaml` and is shared across all overlays. That is the right place for it unless an overlay needs a different catalog channel or source.

```
base/
  subscription.yaml   <-- place it here (shared across all overlays)
```

If you need environment-specific subscriptions (for example, a pre-release channel in dev only):

1. Remove `subscription.yaml` from `base/kustomization.yaml` resources.
2. Create a `subscription.yaml` inside each overlay directory that needs it, and add it to that overlay's `kustomization.yaml` resources list.

The existing `base/subscription.yaml` targets `stable-2.6` from `redhat-operators`. Adjust `channel` and `source` there if you upgrade to a newer stream.

## Secret patterns

Choose one approach and keep plaintext secrets out of Git.

- **Sealed Secrets:** `secrets/sealedsecret-aap-admin-password-prod.yaml`
- **HashiCorp Vault + ESO (used by prod sync):**
  - `overlays/prod/secretstore-vault.yaml`
  - `overlays/prod/externalsecret-aap-admin-password-prod.yaml`

## Notes

- Root `kustomization.yaml` in this folder points to `overlays/prod`.
- Set Argo CD `Application.spec.source.path` to `clusters/openshift/apps/aap-26` for prod, or to `clusters/openshift/apps/aap-26/overlays/<env>` for dev/stage/minimal.
- Verify AAP CR fields against your installed operator version.
- For Vault, update URL, KV path, auth mount, and role in `overlays/prod/secretstore-vault.yaml`.
