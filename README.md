# Kubernetes

Terraform module to deploy a Kubernetes cluster on Azure by using the managed Kubernetes solution AKS. For security reasons it will only deploy a rbac enabled clusters and requires an Azure AD application for authenticating users. This account can be created with the module [avinor/kubernetes-azuread-integration/azurerm](https://github.com/avinor/terraform-azurerm-kubernetes-azuread-integration). Service principal required can be created with [avinor/service-principal/azurerm](https://github.com/avinor/terraform-azurerm-service-principal) module. It is not required to grant the service principal any roles, this module will make sure to grant required roles. That does however mean that the deployment has to run with Owner priviledges.

## Usage

This example deploys a simple cluster with one node pool. The service principal and Azure AD integration secrets need to be changed.

Example uses [tau](https://github.com/avinor/tau) for deployment.

```terraform
module {
    source = "avinor/kubernetes/azurerm"
    version = "1.0.02
}

inputs {
    name = "simple"
    resource_group_name = "simple-aks-rg"
    location = "westeurope"
    service_cidr = "10.0.0.0/24"
    kubernetes_version = "1.13.5"

    service_principal = {
        client_id = "00000000-0000-0000-0000-000000000000"
        client_secret = "00000000-0000-0000-0000-000000000000"
    }

    azure_active_directory = {
        client_app_id = "00000000-0000-0000-0000-000000000000"
        server_app_id = "00000000-0000-0000-0000-000000000000"
        server_app_secret = "00000000-0000-0000-0000-000000000000"
    }

    agent_pools = [
        {
            name = "linux"
            vnet_subnet_id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mygroup1/providers/Microsoft.Network/virtualNetworks/myvnet1"
        },
    ]
}
```

If using tau to deploy the service principal and Azure AD integration secrets too it could be read from dependencies.

```terraform
dependency "service_principal" {
    # Created with azure/service-principal/azurerm module
    source = "./aks-service-principal.hcl"
}

dependency "azuread" {
    # Created with azure/kubernetes-azuread-integration/azurerm module
    source = "./aks-ad-integration.hcl"
}

module {
    source = "avinor/kubernetes/azurerm"
    version = "1.0.02
}

inputs {
    name = "simple"
    resource_group_name = "simple-aks-rg"
    location = "westeurope"
    service_cidr = "10.0.0.0/24"
    kubernetes_version = "1.13.5"

    service_principal = {
        client_id = dependency.service_principal.outputs.client_id
        client_secret = dependency.service_principal.outputs.client_secret
    }

    azure_active_directory = {
        client_app_id = dependency.azuread.outputs.client_app_id
        server_app_id = dependency.azuread.outputs.server_app_id
        server_app_secret = dependency.azuread.outputs.server_app_secret
    }

    agent_pools = [
        {
            name = "linux"
            vnet_subnet_id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mygroup1/providers/Microsoft.Network/virtualNetworks/myvnet1"
        },
    ]
}
```

Similar to this the virtual network subnet id could also be retrieved from a dependency.

## Diagnostics

Diagnostics settings can be sent to either storage account, event hub or Log Analytics workspace. The variable `diagnostics.destination` is the id of receiver, ie. storage account id, event namespace authorization rule id or log analytics resource id. Depending on what id is it will detect where to send. Unless using event namespace the `eventhub_name` is not required, just set to `null` for storage account and log analytics workspace.

Setting `all` in logs and metrics will send all possible diagnostics to destination. If not using `all` type name of categories to send.

## Helm

Cluster come with Helm tiller installed. To change the version it installs set the `tiller_version` variable. Once Helm v3 is released this will be removed and manual installation of tiller will not be required.

## Dashboard

AKS comes with dashboard preinstalled, but currently it does not work well with rbac enabled. It is possible to open the dashboard by running `az aks browse`, but it does not have access to read any resources. This could be resolved by granting the dashboard service account access to read, or enable token authentication on the dashboard. Both requires additional configuration after cluster has been deployed.

## Available version

To get a list of available Kubernetes version in a region run the following command. Replace `westeurope` with region of choice.

```bash
az aks get-versions --location westeurope --query "orchestrators[].orchestratorVersion"
```

## Roles

This module will assign the required roles for cluster. These are based on the [Microsoft documentation](https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal). The variables `container_registries` and `storage_contributor` can be used to grant it access to container registries and storage accounts.

If cluster needs to manage some Managed Identities that can be done by using the input variable `managed_identities`. The AKS service principal will be granted `Managed Identity Operator` role to those identities.

## Service accounts

Using the `service_accounts` variable it is possible to create some default service accounts. For instance to create a service account with `cluster_admin` role that can be used in CI / CI pipelines. It is not recommended to use the admin credentials as they cannot be revoked later.
