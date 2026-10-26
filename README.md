# azure-aks-customer

Your AKS environments. This repo holds **configuration only**. The platform team's [`azure-aks-platform`](https://github.com/my-org/azure-aks-platform) does the provisioning when this repo's workflow calls it.

```
environments/
  prod/
    config.yaml        what to provision and which optional features are on
    rbac/              optional Kubernetes Roles/RoleBindings (*.yaml; examples ship as *.yaml.example)
.github/workflows/
  aks.yml              plans on pull requests, applies on merge to main
```

## Making a change

1. Edit `environments/<env>/config.yaml` on a branch and open a pull request. The workflow runs a plan; read it in the Actions run log.
2. Merge. The apply waits for approval in the GitHub environment `<env>`, then runs.

## Features

Toggle these under `features:` in `config.yaml`. Leave one out to get the platform default.

| Feature | Default |
|---|---|
| `node_auto_provisioning` (managed Karpenter; `false` = cluster-autoscaler on `cluster.user_node_pools`) | `true` |
| `kms_etcd_encryption`, `defender`, `azure_policy`, `key_vault_secrets_provider` | `true` |
| `container_insights`, `managed_prometheus` | `true` |
| `delete_lock` | `true` in prod |
| `keda`, `vpa` | `false` |
| `kyverno` (with the Policy Reporter UI), `reloader` | `true` |
| `rbac_manifests` (apply `rbac/*.yaml`) | `true` |

**Traefik and ArgoCD are always installed.** You configure them under `platform.traefik` and `platform.argocd`, but you can't turn them off.

Every supported key is listed in the [platform config reference](https://github.com/my-org/azure-aks-platform#config-reference-environmentsenvconfigyaml). Unknown or misspelled keys fail the plan.

## Adding an environment

Copy `environments/prod` to `environments/<env>`, edit `config.yaml`, then add a job to `.github/workflows/aks.yml` with `environment: <env>` and that environment's state storage account. Ask the platform team to onboard the new environment (pipeline identity, runner, GitHub environment).

## Upgrading the platform

The workflow pins a platform release twice: in `uses: ...@vX.Y.Z` and in `platform_version: vX.Y.Z`. Change both to the same new tag, read the platform release notes, and open a pull request to see the plan.
