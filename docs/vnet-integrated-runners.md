# Github Hosted Runners with virtual network integration - setup guide

## Background

You can use GitHub-hosted runners in an Azure VNET. This enables you to use GitHub-managed infrastructure for CI/CD while providing you with full control over the networking policies of your runners.

## Relevant resources

* [General Information and supported regions](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization)
* [Github configuration documentation](https://docs.github.com/en/enterprise-cloud@latest/admin/configuration/configuring-private-networking-for-hosted-compute-products/configuring-private-networking-for-github-hosted-runners-in-your-enterprise)

### Prerequisites

Copy necessary files to your github repository

```
.github/workflows/deploy-github-runner-vnet.yaml
.github/workflows/github-runners-network-settings.yaml
services/vnet-integrated-runners
```

Fetch the `databaseId` for the Github Enterprise organization. Required for deployment of the networkSettings resource in Azure. Example:

```shell
curl -H "Authorization: Bearer BEARER_TOKEN" -X POST \
  -d '{ "query": "query($slug: String!) { enterprise (slug: $slug) { slug databaseId } }" ,
        "variables": {
          "slug": "ENTERPRISE_SLUG"
        }
      }' \
https://api.github.com/graphql
```

[Github documentation](https://docs.github.com/en/enterprise-cloud@latest/admin/configuration/configuring-private-networking-for-hosted-compute-products/configuring-private-networking-for-github-hosted-runners-in-your-enterprise#1-obtain-the-databaseid-for-your-enterprise)

### 1. Deploy necessary Azure resources

In order to setup a Github hosted runner with vnet integration, a vnet with a subnet + a specific NSG and a network settings resource must be deployed first. The network settings resource will later be used to create a link between Github enterprise organization and the subnet in Azure.

#### Bicep resource deployment

Adjust the `vnetAddressPrefix` and `snetAddressPrefix` parameters in the `services\vnet-integrated-runners\parameters\vnet.bicepparam` file based on your requirements for the IP range. Set a value for the `baseName` parameter.

* [Overview of IP per team](https://github.com/techcloud0/architecture-platform/blob/main/docs/required/AzureIpScheme-BIDBAX.md)

The deployment will create a NSG that will be attached to the subnet. The NSG includes all the rules required a Github with an addition of the `AllowOutboundAzureCloud` rule, that allows outbound traffic towards AzureCloud. This is necessary for the SPN that will be used for deployments running on this Github runner to be able to authenticate, as well as for the AzCli tasks to work. VWAN integration can also be added if necessary.

#### Workflow configuration

`.github/workflows/github-runners-network-settings.yaml` is a custom workflow that is required, as the networkSettings resource lacks a bicep/ARM reference, and has to be deployed with cli. No changes to this file are necessary.

`.github\workflows\deploy-github-runner-vnet.yaml` needs input for the deployment jobs. Team-orange example:

```yaml
jobs:
  deploy-vnet:
    uses: techcloud0/platform-reusable-workflows/.github/workflows/bicep-deploy-sub.yaml@main
    with:
      azure_tenant_id: <tenantid>
      azure_client_id: <spn-client-id>
      azure_subscription_id: <subscription-id>
      location: norwayeast
      base_name: bai-c-github-runner
      template_file: services/vnet-integrated-runners/main.bicep
      parameters: >-
        services/vnet-integrated-runners/parameters/vnet.bicepparam

  deploy-network-settings:
    uses: ./.github/workflows/github-runners-network-settings.yaml
    needs: deploy-vnet
    with:
      azure_tenant_id: <tenantid>
      azure_client_id: <spn-client-id>
      azure_subscription_id: <subscription-id>
      location: norwayeast
      runner_rg_name: ${{ fromJson(needs.deploy-vnet.outputs.output).resourceGroupName.value }}
      network_settings_resource_name: bai-c-github-runner-network-settings
      subnet_id: ${{ fromJson(needs.deploy-vnet.outputs.output).subnetId.value }}
      github_database_id: <databaseId-from-prerequisites>
```

The second job, `deploy-network-settings` will output an ID that belongs to the networkSettings resources. Save that ID for the next step.

### 2. Configuration in Github

Configuration in Github can only be done by users with elevated privileges on the enterprise level.

*[Steps 1, 2 and 3 from this document must be completed. Input the networkSettings ID from the deployment in step 1](https://docs.github.com/en/enterprise-cloud@latest/admin/configuration/configuring-private-networking-for-hosted-compute-products/configuring-private-networking-for-github-hosted-runners-in-your-enterprise#creating-a-network-configuration-for-your-enterprise-in-github)

### 3. Use the vnet integrated Github runner

With the above steps completed, the runner should now be ready to use. To send jobs to this runner, add `runs-on: <runner-name>` to your workflow yaml. Example:

```yaml
jobs:
  job-example:
    runs-on: <runner-name>
```

If your deployment uses a reusable workflow, the `runs-on` flag needs to be added in the reusable workflow itself.
