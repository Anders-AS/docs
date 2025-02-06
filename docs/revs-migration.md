# Revenue Streams platform migration

## Background

The product team Revenue Streams is migrating their existing applications from the Soteria platform to the new platform developed by the Platform Team. The test environment migration is completed, and the production environment migration is scheduled for 17.06.2024.

## Repositories

**K8s manifests:** [wos-system](https://github.com/techcloud0/wos-system)

**Note:** Flux syncs all environments with the main branch instead of promoting test > dev > staging > prod.

**Azure resources deployed to platform subscriptions:** [revs-infra](https://github.com/techcloud0/revs-infra)

**Azure resources deployed to CSP subscriptions:** [infrastructure-Fakturering](https://dev.azure.com/techcloud0/Fakturering/_git/infrastructure-Fakturering)

**Note:** Resources deployed to `BAI Fakturering Test - Basefarm CSP` and `BAI Fakturering Production - Basefarm CSP`, including the SQL Server and storage account the applications communicate with.

**Application repos:** [fakturering-api](https://github.com/techcloud0/fakturering-api), [bax-invoicing (invoicing-kredinor)](https://github.com/techcloud0/bax-invoicing), and [invoicing-portal](https://github.com/techcloud0/invoicing-portal)

**Note:** Federated credentials are provisioned from the application repos. Deployment using ADO pipelines.

## Access

Write permissions to the above repositories and contributor on:

* `revs-core - Team Revenue Streams - Core`
* `revs-test - Team Revenue Streams - Test`
* `revs-prod - Team Revenue Streams - Prod`

Member of the Entra ID group Team Platform Enabling for cluster admin access.

### Local connection with kubectl and flux CLI

**1.** Activate `Directory Reader` in PIM.

**2.** Add yourself as cluster admin: `revs-prod-nwe-platform-aks` > Cluster configuration > Cluster admin ClusterRoleBinding > Edit chosen groups > add Team Platform Enabling > Select > Apply

**3.** Add your IP to authorized IP address ranges with

```bash
RG=revs-prod-nwe-platform-rg
AKSNAME=revs-prod-nwe-platform-aks
SUBSCRIPTION_ID=cfeb162a-f3ec-44f1-978d-17146b636e65
CURRENT_IP=<your-ip-address>

az account set --subscription $SUBSCRIPTION_ID
az aks update --resource-group $RG --name $AKSNAME --api-server-authorized-ip-ranges $CURRENT_IP,4.246.50.40/29,62.101.211.129/32,4.236.252.184/29,62.101.221.46/32,62.101.231.100/32
```

**4.** Verify connection with

```bash
kubectl get ns
flux get source git -A
```

**Note:** If the platform-infra workflow is executed - repeat the above steps to get access again.

## Production migration

### Code changes

[This PR to wos-system](https://github.com/techcloud0/wos-system/pull/19) added the production manifests. The prod resources are not synced with flux yet because they're disabled in [overlays/prod/kustomization.yaml](https://github.com/techcloud0/wos-system/pull/28).

[This PR to fakturering-api](https://github.com/techcloud0/fakturering-api/pull/43) created a new federated credential for managed identity used by the `fakturering-api` application.

[this PR to bax-invoicing](https://github.com/techcloud0/bax-invoicing/pull/92) created a new federated credential for managed identity used by the `invoicing-kredinor` application.

[This PR to infrastructure-Fakturering](https://dev.azure.com/techcloud0/Fakturering/_git/infrastructure-Fakturering/pullrequest/784) links private endpoints in `fakturering-prod-rg` (`BAI Fakturering Production - Basefarm CSP`) to new private dns zones in `revs-core-nwe-privatedns-rg` (`revs-core - Team Revenue Streams - Core`). This PR is not merged yet because the previous links have to be deleted first.

[This PR to revs-infra](https://github.com/techcloud0/revs-infra/pull/19) adds required role binding such that `DevopsSPN_fakturering-production-subscription` is allowed to link the private endpoints and private dns zones

[This PR to team-orange](https://github.com/techcloud0/team-orange/pull/439) enabled connectivity between `revs-prod-nwe-platform-vnet` and `fakturering-prod-vnet`

[This PR to wos-system](https://github.com/techcloud0/wos-system/pull/26) added SSL certificates.

[bax-invoicing/pull/93](https://github.com/techcloud0/bax-invoicing/pull/93), [fakturering-api/pull/42](https://github.com/techcloud0/fakturering-api/pull/42), [invoicing-portal/pull/193](https://github.com/techcloud0/invoicing-portal/pull/193) changed the application pipelines to push images to `revscorenweregistrycr` (`revs-core - Team Revenue Streams - Core`) instead of `soteriaprodacr` (`BAI Platform PCI VCE Production - Basefarm CSP`).

### Migration steps

The applications cannot run in `soteria-prod-green-aks` and `revs-prod-nwe-platform-aks` at the same time because they communicate with the same database and storage account. The private endpoints can only be linked to a single private dns zone in each namespace.

#### Scale down deployments in soteria-prod-blue-aks

**1.** Connect to management-vm and set context to soteria-prod-green-aks.

```bash
kubectl scale deployment bax-fakturering-api --replicas=0 -n fakturering
kubectl scale deployment invoicing-kredinor-app --replicas=0 -n fakturering
kubectl scale deployment invoicing-portal-app --replicas=0 -n fakturering
```

**Note:** Deployment of these applications to Soteria is disabled in the CD pipelines, so they will not be redeployed.

#### Link private endpoints to platform private dns zones

**2.** Activate Contributor on `BAI Fakturering Production - Basefarm CSP` in PIM.

**3.** `BAI Fakturering Production - Basefarm CSP` > Resource groups > `fakturering-prod-rg` > `bai-prod-fakturering-sql-pep` > DNS configuration > Delete `privatelink_database_windows_net`.

**4.** `BAI Fakturering Production - Basefarm CSP` > Resource groups > `fakturering-prod-rg` > `baiprodfakturering-pep` > DNS configuration > Delete `privatelink_blob_core_windows_net`.

**5.** Merge [this PR](https://dev.azure.com/techcloud0/Fakturering/_git/infrastructure-Fakturering/pullrequest/784) to link private endpoints to new private dns zones.

**6.** Run [this pipeline](https://dev.azure.com/techcloud0/Fakturering/_build?definitionId=79) to deploy the changes.

#### Sync production manifests to revs-prod-nwe-platform-aks

**7.** Merge [this draft PR](https://github.com/techcloud0/wos-system/pull/28) to enable flux sync for prod manifests.

**8.** Verify that everything works with

```bash
flux get source git -A
flux get ks -A
kubectl get pods -n wos
```

**9.** AMS can now switch the URL for their application that consumes `fakturering-api` to `fakturering-api.wos.revs-stoe.cloud/fakturering-api`.
