# Workload Identity migration guide

## Background

AAD Pod Identity is archived and no longer maintained after 12.10.23. The Soteria Vipps clusters are scheduled to migrate to Azure Workload Identity, the new recommended approach for authenticating workloads to access Azure resources. The migration is bundled together with the AKS upgrade from 1.26 to 1.28 in November. This allows us to verify that authentication works for all product teams before switching the active environment to the new blue-green deployment.

## Relevant resources

* [Migrate from pod managed-identity to workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-migrate-from-pod-identity)
* [Use Microsoft Entra Workload ID with Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview?tabs=dotnet)

## Product team migration guide

The following steps outline the responsibility of product teams to ensure a successful migration. It is possible to utilize the existing Managed Identities already used with AAD Pod Identity. However, some changes are required in order to use these identities with Workload Identity as a replacement for the AAD Pod Identity CRDs.

### Prerequisites

It is recommended to use the latest version of the Azure Identity SDK for all applications. After following this migration guide pods will automatically switch to using Workload Identity after a restart if your application is using the latest version of the Azure Identity SDK. If your application does not use the latest version, and it is not an option to upgrade, a temporary alternative is to use [Microsofts migration sidecar](https://learn.microsoft.com/en-us/azure/aks/workload-identity-migrate-from-pod-identity#migrate-from-older-version) which proxies the IMDS transactions to OIDC.

### 1. Create Federated Identity Credentials

Create federated identity credentials for each existing managed identity. One credential should be created for each service account the managed identity will be used with. The AKS cluster is imported using the `existing` keyword to access the OICD issuer profile URL `aks.properties.oidcIssuerProfile.issuerURL` added to the `issuer` property of the federated credential. `<service-account-namespace>` and `<service-account-name>` refers to the namespace and name of the service account created in the next step.

```bicep
resource aks 'Microsoft.ContainerService/managedClusters@2022-05-02-preview' existing = {
  name: '<soteria-cluster-name>'
  scope: resourceGroup('<soteria-cluster-subscription-id>', '<soteria-cluster-resource-group-name>')
}

resource managedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '<managed-identity-name>'
  ...
}

resource federatedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials@2023-01-31' = {
  name: '<federated-identity-credential-name>'
  properties: {
    audiences: [
      'api://AzureADTokenExchange'
    ]
    issuer: aks.properties.oidcIssuerProfile.issuerURL
    subject: 'system:serviceaccount:<service-account-namespace>:<service-account-name>'
  }
  parent: managedIdentity
}
```

### 2. Create Service Accounts

Create a Service Account for each federated credential. This is an example k8s manifest where `<client-id>` is the Managed Identity ID, and `<service-account-name>` refers to the same name referred to when creating the federated credential.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: <client-id>
  name: <service-account-name>
  namespace: <service-account-namespace>

```

### 3. Add required label to workloads

Add required label `azure.workload.identity/use: "true"` to the pod spec of k8s workloads accessing Azure resources using Workload Identity. Example pod manifest accessing a secret in Key Vault, described by `<key-vault-url>`, `<key-vault-name>`, and `<secret-name>`. This example assumes the Managed Identity is added to the Key Vault access policies. The value of secret `<secret-name>` is printed to the pod log to verify key vault authentication (assuming the secret exists in key vault).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: workload-identity-demo
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: workload-identity-sa
  containers:
    - image: ghcr.io/azure/azure-workload-identity/msal-go
      name: oidc
      env:
      - name: KEYVAULT_URL
        value: <key-vault-url>
      - name: KEYVAULT_NAME
        value: <key-vault-name>
      - name: SECRET_NAME
        value: <secret-name>
  nodeSelector:
    kubernetes.io/os: linux
```

### 4. Remove old AAD Pod Identity CRDs

After verifying that workload identity authentication works as intended the AAD Pod Identity CRDs can be removed.

## Example

This example runs in the `Demo - Lab` subscription and illustrates how to use Workload Identity to authenticate a Java Spring Boot application running in AKS to an Azure PostgreSQL database.

* [demo-infra](https://github.com/techcloud0/demo-infra/tree/main/bicep/modules/postgres-workload-identity) - Azure resources and documentation
* [platform-demo-app](https://github.com/techcloud0/platform-demo-app/tree/main/JavaApp) - Java Spring Boot application
* [demo-system](https://github.com/techcloud0/demo-system) - Kubernetes manifests
