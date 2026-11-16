# azure-aks-customer

Your AKS environments. This repo holds **configuration only**. The platform team's `azure-aks-platform` does the provisioning when this repo's Azure DevOps pipeline calls it.

```
environments/
  prod/
    config.yaml        what to provision, which optional features are on, and mode (create / destroy)
    rbac/              optional Kubernetes Roles/RoleBindings (*.yaml; examples ship as *.yaml.example)
azure-pipelines.yml    Azure DevOps pipeline: plans on every run, applies or destroys on main
.github/workflows/
  aks.yml              GitHub Actions alternative (create mode only)
```

## How the pipeline runs

Every commit to `main` that changes `environments/` or `azure-pipelines.yml` starts the pipeline. Pull requests run the plan only.

```
commit ─▶ Plan_<env> ──┬─ mode: create  ─▶ Apply_<env>    approval on environment <env>
                       │                                   01-cluster, then 02-platform
                       └─ mode: destroy ─▶ Destroy_<env>  "Confirm destroy" + approval on environment <env>-destroy
                                                           02-platform, then 01-cluster
```

- **Plan_\<env>** checks `config.yaml`, then plans both stacks. The run summary shows the mode and which stacks would change. The full plan is in the log of the *Terraform plan* step.
- **Apply / Destroy** run only on `main`, and only when the plan has changes. A commit that changes only `dev` doesn't ask anyone to approve `prod`.
- **Plan only:** start the pipeline by hand and tick *Plan only*.

## The create / destroy toggle

`mode` at the top of `config.yaml` controls everything the pipeline provisions in that environment.

| `mode` | What a commit to `main` does |
|---|---|
| `create` (default) | Provisions the environment and keeps it in sync with `config.yaml`. Leave it here day to day; every change you commit is planned and applied. |
| `destroy` | Deletes everything this pipeline created in the environment: the add-ons (02-platform), then the cluster, its identities, KMS key vault, monitoring and the resource group, if the pipeline created it (01-cluster). |

### Destroying an environment

1. On a branch, set both of these in `environments/<env>/config.yaml`:
   ```yaml
   mode: destroy
   features:
     delete_lock: false   # required; destroy is refused while the delete lock is on (it is on by default in prod)
   ```
2. Open a pull request. The plan stage shows the **destroy plan**, so reviewers see exactly what will be deleted.
3. Merge. `Destroy_<env>` waits for:
   - **Confirm destroy:** one of the `destroyApprovers` must resume it. The person who started the run can't approve it.
   - Any approvals and checks on the ADO environment `<env>-destroy`.
4. Then it destroys 02-platform and 01-cluster, in that order.

The pipeline won't touch anything else. The VNet, the state storage account, the platform Key Vault and the ACR are inputs, not outputs, so they stay. Once an environment is destroyed, more commits in `destroy` mode do nothing, because the plan has no changes.

### Re-creating it

Set `mode: create` (and `delete_lock: true` again for prod) and merge. The environment is provisioned from scratch with the same names. The exception is the KMS key vault, which gets a new random suffix. It has purge protection, so the old vault stays soft-deleted for 90 days and can't be purged early.

## Making a change

1. Edit `environments/<env>/config.yaml` on a branch and open a pull request. Read the plan in the pipeline run.
2. Merge. `Apply_<env>` waits for approval on the ADO environment `<env>`, then runs.

## Features

Toggle these under `features:` in `config.yaml`. Leave one out to get the platform default.

| Feature | Default |
|---|---|
| `node_auto_provisioning` (managed Karpenter; `false` = cluster-autoscaler on `cluster.user_node_pools`) | `true` |
| `kms_etcd_encryption`, `defender`, `azure_policy`, `key_vault_secrets_provider` | `true` |
| `container_insights`, `managed_prometheus` | `true` |
| `delete_lock` (must be `false` for `mode: destroy`) | `true` in prod |
| `keda`, `vpa` | `false` |
| `kyverno` (with the Policy Reporter UI), `reloader` | `true` |
| `rbac_manifests` (apply `rbac/*.yaml`) | `true` |

**Traefik and ArgoCD are always installed.** You configure them under `platform.traefik` and `platform.argocd`, but you can't turn them off.

Every supported key is listed in the platform README's config reference. Unknown or misspelled keys fail the plan.

## Adding an environment

Copy `environments/prod` to `environments/<env>` and edit `config.yaml`. Then add a `- template: pipelines/provision.yml@platform` block for it in `azure-pipelines.yml`, with its own service connection, state storage account and `stateKeyPrefix`. Ask the platform team to onboard the new environment: service connection, agent pool access, and the ADO environments `<env>` and `<env>-destroy`.

## Upgrading the platform

Change `ref: refs/tags/vX.Y.Z` under `resources.repositories` in `azure-pipelines.yml`. That one ref pins both the pipeline template and the Terraform code. Read the platform release notes, then open a pull request to see the plan.

## GitHub Actions

`.github/workflows/aks.yml` is kept for repos hosted on GitHub. It supports `mode: create` only and fails when the mode is `destroy`. Don't run both pipelines against the same environment. Delete the workflow if this repo lives in Azure Repos.
