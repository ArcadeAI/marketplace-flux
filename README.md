# marketplace-flux

Bootstrap and chart-install manifests consumed by the Marketplace Managed App's `Microsoft.KubernetesConfiguration/fluxConfigurations` resource.

## Status

**Staged in monorepo as a starting point.** The eventual home is a separate public repo (`github.com/ArcadeAI/marketplace-flux`) versioned in lockstep with chart releases — `fluxConfigurations.gitRepository.repositoryRef.tag` will pin to a chart-version tag.

This directory should be moved out before the next Marketplace publish iteration. Until then, the Bicep `fluxConfigurations` resource won't have a real Git URL to point at.

## Layout

```
marketplace-flux/
├── bootstrap/          # Reconciled FIRST (depends on nothing)
│   ├── cert-manager-helmrelease.yaml
│   ├── ingress-nginx-helmrelease.yaml
│   ├── external-secrets-helmrelease.yaml
│   ├── letsencrypt-prod-clusterissuer.yaml   # ${LETSENCRYPT_EMAIL}
│   ├── arcade-kv-secretstore.yaml            # ${KEY_VAULT_NAME}, ${ESO_CLIENT_ID}, ${TENANT_ID}
│   └── arcade-externalsecrets.yaml           # materializes the secret names the chart expects
│
└── arcade/             # Reconciled AFTER bootstrap (kustomizations.arcade.dependsOn: [bootstrap])
    └── arcade-helmrelease.yaml               # ${HOSTNAME}, ${POSTGRES_HOST}, etc.
```

## Variable substitution

Flux's `postBuild.substitute` and `postBuild.substituteFrom` resolve `${VAR}` placeholders in the YAML at reconciliation time.

- **Non-secret values** (HOSTNAME, NAMESPACE, RELEASE_NAME, POSTGRES_HOST, KEY_VAULT_NAME, ESO_CLIENT_ID, TENANT_ID, LETSENCRYPT_EMAIL, CHART_VERSION) come from `fluxConfigurations.kustomizations.<name>.postBuild.substitute` in the Bicep template.
- **Secrets** (POSTGRES_PASSWORD, ENTRA_APP_CLIENT_SECRET, PLAYGROUND_API_KEY) come from `fluxConfigurations.configurationProtectedSettings`, which the platform auto-materializes as a Secret named `<fluxConfigName>-config-protected-settings` in `flux-system`. The `arcade-helmrelease.yaml` Kustomization references this Secret via `postBuild.substituteFrom`.

If the auto-materialization assumption is wrong (load-bearing Q1 in `capsol:guidance://migrate-helm-deploy-to-k8s-extension`), the fallback is: publisher pre-writes the secrets to Key Vault at deploy time; ESO + SecretStore + ExternalSecrets in `bootstrap/` materialize them; the arcade Kustomization references the ESO-materialized Secret instead.

## What's NOT here yet

- The actual YAML files. This README + directory structure is the first scaffold.
- Versioning / tagging strategy (tag in lockstep with `deploy/charts/arcade/Chart.yaml` version).
- CI to validate that every `${VAR}` referenced in YAML has a corresponding `substitute:` entry in the Bicep template.

## Why this lives here today

Iterating on a separate public repo + the Bicep template simultaneously is operationally painful (two PRs per change, two release cycles). Staging the manifests in the monorepo lets us iterate on the design without the publishing surface. Once stable, the directory moves to its own repo and `fluxConfigurations.gitRepository.url` switches.
