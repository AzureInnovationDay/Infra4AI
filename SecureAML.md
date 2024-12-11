
# Workspace Managed Virtual Network Isolation

Azure Machine Learning provides support for managed virtual network (managed virtual network) isolation. Managed virtual network isolation streamlines and automates your network isolation configuration with a built-in, workspace-level Azure Machine Learning managed virtual network. The managed virtual network secures your managed Azure Machine Learning resources, such as compute instances, compute clusters, serverless compute, and managed online endpoints. 

Securing your workspace with a *managed network* provides network isolation for __outbound__ access from the workspace and managed computes. An *Azure Virtual Network that you create and manage* is used to provide network isolation __inbound__ access to the workspace. For example, a private endpoint for the workspace is created in your Azure Virtual Network. Any clients connecting to the virtual network can access the workspace through the private endpoint. When running jobs on managed computes, the managed network restricts what the compute can access.

## Managed Virtual Network Architecture

When you enable managed virtual network isolation, a managed virtual network is created for the workspace. Managed compute resources you create for the workspace automatically use this managed virtual network. The managed virtual network can use private endpoints for Azure resources that are used by your workspace, such as Azure Storage, Azure Key Vault, and Azure Container Registry. 

There are two different configuration modes for outbound traffic from the managed virtual network:

> [!TIP]
> Regardless of the outbound mode you use, traffic to Azure resources can be configured to use a private endpoint. For example, you may allow all outbound traffic to the internet, but restrict communication with Azure resources by adding outbound rules for the resources.

| Outbound mode | Description | Scenarios |
| ----- | ----- | ----- |
| Allow internet outbound | Allow all internet outbound traffic from the managed virtual network. | You want unrestricted access to machine learning resources on the internet, such as python packages or pretrained models.<sup>1</sup> |
| Allow only approved outbound | Outbound traffic is allowed by specifying service tags. | * You want to minimize the risk of data exfiltration, but you need to prepare all required machine learning artifacts in your private environment.</br>* You want to configure outbound access to an approved list of services, service tags, or FQDNs. |
| Disabled | Inbound and outbound traffic isn't restricted or you're using your own Azure Virtual Network to protect resources. | You want public inbound and outbound from the workspace, or you're handling network isolation with your own Azure virtual network. |

1: You can use outbound rules with _allow only approved outbound_ mode to achieve the same result as using allow internet outbound. The differences are:

* You must add rules for each outbound connection you need to allow.
* Adding FQDN outbound rules __increase your costs__ as this rule type uses Azure Firewall. For more information, see [Pricing](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-managed-network?view=azureml-api-2&tabs=azure-cli#pricing)
* The default rules for _allow only approved outbound_ are designed to minimize the risk of data exfiltration. Any outbound rules you add might increase your risk.

The managed virtual network is preconfigured with [required default rules](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-managed-network?view=azureml-api-2&tabs=portal#list-of-required-rules)). It's also configured for private endpoint connections to your workspace, workspace's default storage, container registry, and key vault __if they're configured as private__ or __the workspace isolation mode is set to allow only approved outbound__. After choosing the isolation mode, you only need to consider other outbound requirements you might need to add.

The following diagram shows a managed virtual network configured to __allow internet outbound__:

![image](/media/how-to-managed-network/internet-outbound.svg)

The following diagram shows a managed virtual network configured to __allow only approved outbound__:

> [!NOTE]
> In this configuration, the storage, key vault, and container registry used by the workspace are flagged as private. Since they are flagged as private, a private endpoint is used to communicate with them.

![image](/media/how-to-managed-network/only-approved-outbound.svg)

> [!NOTE]
> Once a managed VNet workspace is configured to __allow internet outbound__, the workspace cannot be reconfigured to __disabled__. Similarily, once a managed VNet workspace is configured to __allow only approved outbound__, the workspace cannot be reconfigured to __allow internet outbound__. Please keep this in mind when selecting the isolation mode for managed VNet in your workspace.


### Azure Machine Learning studio

If you want to use the integrated notebook or create datasets in the default storage account from studio, your client needs access to the default storage account. Create a _private endpoint_ or _service endpoint_ for the default storage account in the Azure Virtual Network that the clients use.

Part of Azure Machine Learning studio runs locally in the client's web browser, and communicates directly with the default storage for the workspace. Creating a private endpoint or service endpoint (for the default storage account) in the client's virtual network ensures that the client can communicate with the storage account.

For more information on creating a private endpoint or service endpoint, see the [Connect privately to a storage account](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints) and [Service Endpoints](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview) articles.

### Secured associated resources

If you add the following services to the virtual network by using either a service endpoint or a private endpoint (disabling the public access), allow trusted Microsoft services to access these services:

| Service | Endpoint information | Allow trusted information |
| ----- | ----- | ----- |
| __Azure Key Vault__| [Service endpoint]([https://learn.microsoft.com/en-us/azure/key-vault/general/overview-vnet-service-endpoints](https://learn.microsoft.com/en-us/azure/key-vault/general/overview-vnet-service-endpoints))</br>[Private endpoint](https://learn.microsoft.com/en-us/azure/key-vault/general/private-link-service) | [Allow trusted Microsoft services to bypass this firewall](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-secure-workspace-vnet?view=azureml-api-2#secure-azure-key-vault) |
| __Azure Storage Account__ | [Service and private endpoint](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-secure-workspace-vnet?view=azureml-api-2&tabs=se#secure-azure-storage-accounts)</br>[Private endpoint](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-secure-workspace-vnet?view=azureml-api-2&tabs=pe#secure-azure-storage-accounts) | [Grant access from Azure resource instances](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security#grant-access-from-azure-resource-instances)</br>__or__</br>[Grant access to trusted Azure services](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security#grant-access-to-trusted-azure-services) |
| __Azure Container Registry__ | [Private endpoint](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-private-link) | [Allow trusted services](https://learn.microsoft.com/en-us/azure/container-registry/allow-access-trusted-services) |

## Prerequisites

Before following the steps in this article, make sure you have the following prerequisites:

### Azure portal

* An Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/).

* The __Microsoft.Network__ resource provider must be registered for your Azure subscription. This resource provider is used by the workspace when creating private endpoints for the managed virtual network.

    For information on registering resource providers, see [Resolve errors for resource provider registration](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/error-register-resource-provider).

* The Azure identity you use when deploying a managed network requires the following [Azure role-based access control (Azure RBAC)](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) actions to create private endpoints:

    * `Microsoft.MachineLearningServices/workspaces/privateEndpointConnections/read`
    * `Microsoft.MachineLearningServices/workspaces/privateEndpointConnections/write`

---

> [!NOTE]
> If you are using UAI workspace please make sure to add the Azure AI Enterprise Network Connection Approver role to your identity. For more information, see [User-assigned managed identity](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-identity-based-service-authentication?view=azureml-api-2).

## Configure a managed virtual network to allow only approved outbound

> [!TIP]
> The managed VNet is automatically provisioned when you create a compute resource. When allowing automatic creation, it can take around __30 minutes__ to create the first compute resource as it is also provisioning the network. If you configured FQDN outbound rules, the first FQDN rule adds around __10 minutes__ to the provisioning time. For more information, see [Manually provision the network](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-managed-network?view=azureml-api-2&tabs=portal#manually-provision-a-managed-vnet).


### Azure portal

* __Create a new workspace__:

    1. Sign in to the [Azure portal](https://portal.azure.com), and choose Azure Machine Learning from Create a resource menu.
    2. Provide the required information on the __Basics__ tab.
       ![image](/media/how-to-managed-network/aml_create_basics.png)
 
    4. From the __Networking__ tab, select __Private with Approved Outbound__.

       ![image](/media/how-to-managed-network/private_outbound_settings.png)
    
    5. Check the **Provision Managed Network (Preview)**
       
    6. Choose a **Basic** firewall SKU 
       
    7. Add an _outbound rule_ for the Azure OpenAI ressource, select __Add user-defined outbound rules__ from the __Networking__ tab. From the __Workspace outbound rules__ sidebar, provide the following information:
    
        * __Rule name__: A name for the rule. The name must be unique for this workspace.
        * __Destination type__: Private Endpoint

        The destination type is __Private Endpoint__, provide the following information:

        * __Subscription__: The subscription that contains the Azure resource you want to add a private endpoint for.
        * __Resource group__: The resource group that contains the Azure resource you want to add a private endpoint for.
        * __Resource type__: The type of the Azure resource. **Microsoft.CognitiveServices/accounts**
        * __Resource name__: The name of the Azure resource. **Your Azure OpenAI ressource**
        * __Sub Resource__: The sub resource of the Azure resource type. **account**

        ![image](/media/how-to-managed-network/outbound_aoai_rule.png)
       
  8. Add 2 news _outbound rule_ for the pypi.org repository, select __Add user-defined outbound rules__ from the __Networking__ tab. From the __Workspace outbound rules__ sidebar, provide the following information:
    
        The destination type is __FQDN__, provide the following information:

        * __Rule Name__: Name of the rule, pypi
        * __FQDN destination__: pypi.org

        The destination type is __FQDN__, provide the following information:

        * __Rule Name__: Name of the rule, pypifiles
        * __FQDN destination__: files.pythonhosted.org
        
    9. Select **__Save__** to save the rules. You can continue using __Add user-defined outbound rules__ to add rules.

    10. Continue creating the workspace as normal.

> [!TIP]
> Azure Machine Learning managed VNet doesn't support creating a private endpoint to all Azure resource types. For a list of supported resources, see the [Private endpoints](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-managed-network?view=azureml-api-2&tabs=portal#private-endpoints)  section.

---

## Create private endpoint for all resources created by AML

You need to create Private Endpoint for :

- The AML Workspace itslef
- Azure Blob Storage
- Azure File Store
- Azure Key Vault

Use the **pe** subnet dedicated in your VNet previously created.

Disable Public Access for the **Storage Account and the Key Vault**.

## Create a compute instance

To be able to use a Notebook and execute some code, you need to create a compute instance in the managed VNet.

1. Create a compute, set a **Name** and choose an **SKU**
![image](/media/how-to-managed-network/create_compute_sku.png)

2. Set the **Security** parameters as shown
![image](/media/how-to-managed-network/create_compute_security.png)

3. For the **Applications** and **Tags** category, let the default parameters
4. Select **Review and Create** and **Create**

## Upload a Notebook file

1. In the notebook section, choose (+) to upload a notebook file
2. Choose the *demo.ipynb* file in the **samples** directoty of this repo

## Give permission to your compute instance to use your Azure OpenAI ressource

1. On the Azure OpenAi ressources, give the Role **Cognitive Service OpenAI User**
2. Select a user,group,service principal
3. Search with this pattern [*aml-workspace-name*]/compute/[*compute-vm-name*]
4. Apply the Role

## Let's execute some code

![image](/media/how-to-managed-network/notebook_run.png)

1. Execute the first cell to import python package
2. Replace the **endpoint** value with your own Azure OpenAI endpoint
3. If needed, adapt the deployment name too
4. Execute the second cell to make a call to openai for a chat completion


