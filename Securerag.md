# Network and access configuration for Azure OpenAI On Your Data

Use this guide to learn how to configure networking and access when using Azure OpenAI On Your Data with Microsoft Entra ID role-based access control, virtual networks, and private endpoints.

## Data ingestion architecture 

When you use Azure OpenAI On Your Data to ingest data from Azure blob storage, local files or URLs into Azure AI Search, the following process is used to process the data.

![image](/media/use-your-data/ingestion-architecture.png)


* Steps 1 and 2 are only used for file upload.
* Downloading URLs to your blob storage is not illustrated in this diagram. After web pages are downloaded from the internet and uploaded to blob storage, steps 3 onward are the same.
* Two indexers, two indexes, two data sources and a [custom skill](/azure/search/cognitive-search-custom-skill-interface) are created in the Azure AI Search resource.
* The chunks container is created in the blob storage.
* If the schedule triggers the ingestion, the ingestion process starts from step 7.
*  Azure OpenAI's `preprocessing-jobs` API implements the [Azure AI Search customer skill web API protocol](/azure/search/cognitive-search-custom-skill-web-api), and processes the documents in a queue. 
* Azure OpenAI:
    1. Internally uses the first indexer created earlier to crack the documents.
    1. Uses a heuristic-based algorithm to perform chunking. It honors table layouts and other formatting elements in the chunk boundary to ensure the best chunking quality.
    1. If you choose to enable vector search, Azure OpenAI uses the selected embedding setting to vectorize the chunks.
* When all the data that the service is monitoring are processed, Azure OpenAI triggers the second indexer.
* The indexer stores the processed data into an Azure AI Search service.

For the managed identities used in service calls, only system assigned managed identities are supported. User assigned managed identities aren't supported.

## Inference architecture

![image](/media/use-your-data/inference-architecture.png)

When you send API calls to chat with an Azure OpenAI model on your data, the service needs to retrieve the index fields during inference to perform fields mapping. Therefore the service requires the Azure OpenAI identity to have the `Search Service Contributor` role for the search service even during inference.

If an embedding dependency is provided in the inference request, Azure OpenAI will vectorize the rewritten query, and both query and vector are sent to Azure AI Search for vector search.


## Resource configuration

Use the following sections to configure your resources for optimal secure usage. Even if you plan to only secure part of your resources, you still need to follow all the steps. 

This article describes network settings related to disabling public network for Azure OpenAI resources, Azure AI search resources, and storage accounts. Using selected networks with IP rules is not supported, because the services' IP addresses are dynamic.

## Create resource group

Create a resource group, so you can organize all the relevant resources. Choose the name you like and be sure to choose "France central" as a location.

The resources in the resource group include but are not limited to:
* One Virtual network
* Three key services: one Azure OpenAI, one Azure AI Search, one Storage Account
* Three Private endpoints, each is linked to one key service
* Three Network interfaces, each is associated with one private endpoint
* One Azure Bastion, for the access from on-premises client machines through a Jumphost
* A jumphost VM to access privately to your key services and Web App
* One Web App with virtual network integrated
* Multiple Private DNS zone, so the Web App finds the IP of your Azure OpenAI, and you can connect from a jumphost VM to your Cognitive Services

## Create virtual network

The virtual network has four subnets. 

1. The first subnet is used for the virtual machine.
2. The second subnet is used for the private endpoints for the three key services.
3. The third subnet is empty, and used for Web App outbound virtual network integration.
4. The fourth subnet is the Bastion Subnet

In this lab, when asked to create a virtual network

- Choose the existing resource group
- Name it "vnet-lab"
- Choose "France Central" as location
- Add Four Subnets:
    - "pe" : dedicated to the private endpoints, using the 10.0.0.0/24 prefix
    - "AzureBastionSubnet" : dedicated to the Bastion using the 10.0.1.0/26 prefix
    - "appservice" : dedicated to the Web app Service using the 10.0.2.0/24 prefix
    - "vms" : dedicated to the jumphost VM using the 10.0.1.64/27 prefix
    
To create a virtual network, you can refer to this documentation : https://learn.microsoft.com/en-us/azure/virtual-network/quick-create-portal.

The following procedure creates a virtual network with a resource subnet, an Azure Bastion subnet, and an Azure Bastion host.

1. In the portal, search for and select **Virtual networks**.

2. On the **Virtual networks** page, select **+ Create**.

3. On the **Basics** tab of **Create virtual network**, enter or select the following information:

    | Setting | Value |
    |---|---|
    | **Project details** |  |
    | Subscription | Select your subscription. |
    | Resource group | Select **your existing resource group**. </br> Select **OK**. |
    | **Instance details** |  |
    | Name | Enter **vnet-lab**. |
    | Region | Select **France Central**. |

![image](/media/use-your-data/creation_vnet.png)

4. Select **Next** to proceed to the **Security** tab.

5. Select **Enable Bastion** in the **Azure Bastion** section of the **Security** tab.

    Azure Bastion uses your browser to connect to VMs in your virtual network over secure shell (SSH) or remote desktop protocol (RDP) by using their private IP addresses. The VMs don't need public IP addresses, client software, or special configuration. For more information about Azure Bastion, see [Azure Bastion](../articles/bastion/bastion-overview.md)

    >[!NOTE]
    >[!INCLUDE [Pricing](~/reusable-content/ce-skilling/azure/includes/bastion-pricing.md)]

6. Enter or select the following information in **Azure Bastion**:

    | Setting | Value |
    |---|---|
    | Azure Bastion host name | Enter **bastion**. |
    | Azure Bastion public IP address | Select **Create a public IP address**. </br> Enter **public-ip** in Name. </br> Select **OK**. |

![image](/media/use-your-data/create_bastion.png)

7. Select **Next** to proceed to the **IP Addresses** tab.
    
8. In the address space box in **Subnets**, select the **default** subnet.

9. In **Edit subnet**, enter or select the following information:

    | Setting | Value |
    |---|---|
    | **Subnet details** |  |
    | Subnet template | Leave the default **Default**. |
    | Name | Enter **pe**. |
    | Starting address | Leave the default of **10.0.0.0**. |
    | Subnet size | Leave the default of **/24(256 addresses)**. |

10. Select **Save**.

11. Add the Web App Subnet, select **+Add a subnet**

    | Setting | Value |
    |---|---|
    | **Subnet details** |  |
    | Subnet template | Leave the default **Default**. |
    | Name | Enter **appservices**. |
    | Starting address | Leave the default of **10.0.2.0**. |
    | Subnet size | Leave the default of **/24(256 addresses)**. |

12. Select **Save**.

13. Add the VMs Subnet, select **+Add a subnet**

    | Setting | Value |
    |---|---|
    | **Subnet details** |  |
    | Subnet template | Leave the default **Default**. |
    | Name | Enter **vms**. |
    | Starting address | Leave the default of **10.0.1.64**. |
    | Subnet size | Leave the default of **/27(32 addresses)**. |

14. Select **Save**.

![image](/media/use-your-data/vnet_subnet.png)

15. Select **Review + create** at the bottom of the screen, and when validation passes, select **Create**.

## Configure Azure OpenAI

### Use Sweden Central

During creation of Azure OpenAI resource, choose **Sweden Cental**

### Enable managed identity

To allow your Azure AI Search and Storage Account to recognize your Azure OpenAI Service via Microsoft Entra ID authentication, you need to assign a managed identity for your Azure OpenAI Service. The easiest way is to toggle on system assigned managed identity on Azure portal.

![image](/media/use-your-data/openai-managed-identity.png)

To set the managed identities via the management API, see [the management API reference documentation](/rest/api/aiservices/accountmanagement/accounts/update#identity).

```json

"identity": {
  "principalId": "<YOUR-PRINCIPAL-ID>",
  "tenantId": "<YOUR-TENNANT-ID>",
  "type": "SystemAssigned, UserAssigned", 
  "userAssignedIdentities": {
    "/subscriptions/<YOUR-SUBSCIRPTION-ID>/resourceGroups/my-resource-group",
    "principalId": "<YOUR-PRINCIPAL-ID>", 
    "clientId": "<YOUR-CLIENT-ID>"
  }
}
```

### Enable trusted service

To allow your Azure AI Search to call your Azure OpenAI `preprocessing-jobs` as custom skill web API, while Azure OpenAI has no public network access, you need to set up Azure OpenAI to bypass Azure AI Search as a trusted service based on managed identity. Azure OpenAI identifies the traffic from your Azure AI Search by verifying the claims in the JSON Web Token (JWT). Azure AI Search must use the system assigned managed identity authentication to call the custom skill web API. 

Set `networkAcls.bypass` as `AzureServices` from the management API. For more information, see [Virtual networks article](/azure/ai-services/cognitive-services-virtual-networks?tabs=portal#grant-access-to-trusted-azure-services-for-azure-openai).


### Disable public network access

You can disable public network access of your Azure OpenAI resource in the Azure portal. 

To allow access to your Azure OpenAI Service from your client machines, like using Azure OpenAI Studio, you need to create [private endpoint connections](/azure/ai-services/cognitive-services-virtual-networks?tabs=portal#use-private-endpoints) that connect to your Azure OpenAI resource. Thsi private endpoint will be used by the WebApp, so **it has to be created in France Central**.


## Configure Azure AI Search

You can use basic pricing tier and higher for the search resource. It's not necessary, but if you use the S2 pricing tier, [advanced options](#create-shared-private-link) are available.

### Enable managed identity

To allow your other resources to recognize the Azure AI Search using Microsoft Entra ID authentication, you need to assign a managed identity for your Azure AI Search. The easiest way is to toggle on the system assigned managed identity in the Azure portal.

![image](/media/use-your-data/outbound-managed-identity-ai-search.png)

### Enable role-based access control
As Azure OpenAI uses managed identity to access Azure AI Search, you need to enable role-based access control in your Azure AI Search. To do it on Azure portal, select **Both** or **Role-based access control** in the **Keys** tab in the Azure portal.

![image](/media/use-your-data/managed-identity-ai-search.png)

For more information, see the [Azure AI Search RBAC article](/azure/search/search-security-enable-roles).

### Disable public network access

You can disable public network access of your Azure AI Search resource in the Azure portal. 

To allow access to your Azure AI Search resource from your client machines, like using Azure OpenAI Studio, you need to create [private endpoint connections](/azure/search/service-create-private-endpoint) that connect to your Azure AI Search resource.


### Enable trusted service

You can enable trusted service of your search resource from Azure portal.

Go to your search resource's network tab. With the public network access set to **disabled**, select **Allow Azure services on the trusted services list to access this search service.**

![image](/media/use-your-data/search-trusted-service.png)

You can also use the REST API to enable trusted service. This example uses the Azure CLI and the `jq` tool.

```bash
rid=/subscriptions/<YOUR-SUBSCRIPTION-ID>/resourceGroups/<YOUR-RESOURCE-GROUP>/providers/Microsoft.Search/searchServices/<YOUR-RESOURCE-NAME>
apiVersion=2024-03-01-Preview
#store the resource properties in a variable
az rest --uri "https://management.azure.com$rid?api-version=$apiVersion" > search.json

#replace bypass with AzureServices using jq
jq '.properties.networkRuleSet.bypass = "AzureServices"' search.json > search_updated.json

#apply the updated properties to the resource
az rest --uri "https://management.azure.com$rid?api-version=$apiVersion" \
    --method PUT \
    --body @search_updated.json

```

## Configure Storage Account

### Enable trusted service

To allow access to your Storage Account from Azure OpenAI and Azure AI Search, you need to set up Storage Account to bypass your Azure OpenAI and Azure AI Search as [trusted services based on managed identity](/azure/storage/common/storage-network-security?tabs=azure-portal#trusted-access-based-on-a-managed-identity).

In the Azure portal, navigate to your storage account networking tab, choose "Selected networks", and then select **Allow Azure services on the trusted services list to access this storage account** and click Save.

### Disable public network access

You can disable public network access of your Storage Account in the Azure portal. 

To allow access to your Storage Account from your client machines, like using Azure OpenAI Studio, you need to create [private endpoint connections](/azure/storage/common/storage-private-endpoints) that connect to your blob storage.

## Role assignments

So far you have already setup each resource work independently. Next you need to allow the services to authorize each other.

|Role| Assignee | Resource | Description |
|--|--|--|--|
| `Search Index Data Reader` | Azure OpenAI | Azure AI Search | Inference service queries the data from the index. |
| `Search Service Contributor` | Azure OpenAI | Azure AI Search | Inference service queries the index schema for auto fields mapping. Data ingestion service creates index, data sources, skill set, indexer, and queries the indexer status. |
| `Storage Blob Data Contributor` | Azure OpenAI | Storage Account | Reads from the input container, and writes the preprocessed result to the output container. |
| `Cognitive Services OpenAI Contributor` | Azure AI Search | Azure OpenAI | Custom skill. |
| `Storage Blob Data Reader` | Azure AI Search | Storage Account | Reads document blobs and chunk blobs. |
| `Reader` | Azure AI Foundry Project | Azure Storage Private Endpoints (Blob & File) | Read search indexes created in blob storage within an AI Foundry Project. |
| `Cognitive Services OpenAI User` | Web app | Azure OpenAI | Inference. |


In the above table, the `Assignee` means the system assigned managed identity of that resource.

The admin needs to have the `Owner` role on these resources to add role assignments.

See the [Azure RBAC documentation](/azure/role-based-access-control/role-assignments-portal) for instructions on setting these roles in the Azure portal. You can use the [available script on GitHub](https://github.com/microsoft/sample-app-aoai-chatGPT/blob/main/scripts/role_assignment.sh) to add the role assignments programmatically.

To enable the developers to use these resources to build applications, the admin needs to add the developers' identity with the following role assignments to the resources.

|Role| Resource | Description |
|--|--|--|
| `Cognitive Services OpenAI Contributor` | Azure OpenAI | Call public ingestion API from Azure OpenAI Studio. The `Contributor` role is not enough, because if you only have `Contributor` role, you cannot call data plane API via Microsoft Entra ID authentication, and Microsoft Entra ID authentication is required in the secure setup described in this article. |
| `Cognitive Services User` | Azure OpenAI | List API-Keys from Azure OpenAI Studio.|
| `Contributor` | Azure AI Search | List API-Keys to list indexes from Azure OpenAI Studio.|
| `Contributor` | Storage Account | List Account SAS to upload files from Azure OpenAI Studio.|
| `Contributor` | The resource group or Azure subscription where the developer need to deploy the web app to | Deploy web app to the developer's Azure subscription.|
| `Role Based Access Control Administrator` | Azure OpenAI | Permission to configure the necessary role assignment on the Azure OpenAI resource. Enables the web app to call Azure OpenAI. |

## Configure the jumphost VM

To access the Azure OpenAI Service from your on-premises client machines, one of the approaches is to use a Jump host VM with a Bastion.

1. Create a new VM :

    | Setting | Value |
    |---|---|
    | **Project details** |  |
    | Resource Group | **Your actual resource group** |
    | **Instance details** |  |
    | Virtual Machine Name | **jumphost**. |
    | Region | **France Central**. |
    | Availability Options | **No Infrastructure redundancy required**. |
    | Image | **Windows 10 Pro, version 22H2, x64 Gen2**. |
    | Region | **France Central**. |
    | Size | **Standard D2s_v3**. |
    | **Administrator Account** |  |
    | Username | **adminuser**. |
    | Password | **Choose one**. |
    | Confirm Password | **Choose one**. |
    | **Inbound port rules** |  |
    | Public Inbound Port | **None**. |
    | **Lincensing** |  |
    | Confirm licensing | **Check**. |

![image](/media/use-your-data/create_vm.png)
     
2. Select **Next: Disks** and leave everything as default

![image](/media/use-your-data/vm_disks.png)

3. Select **Next: Networking**

    | Setting | Value |
    |---|---|
    | **Network Interface** |  |
    | Virtual Network | **vnet-lab** |
    | Subnet | **vms** |
    | Public IP | **None** |
   
![image](/media/use-your-data/vm_network.png)

4. Select **Review + Create** and **Create** after validation

## Connect to the Jumphost VM

1. Select your VM and go to **Bastion** section

![image](/media/use-your-data/bastion_connect.png)
 
2. Use the **Username** and **Password** used during VM creation to login into the VM in RDP
   
3. In the VM connet to the azure portal with the login/password of your Azure Pass

## Azure OpenAI Studio

You should be able to use all Azure OpenAI Studio features, including both ingestion and inference, from your on-premises client machines.

## Deploy the 2 models

You will need to deploy 2 models in Sweden Central
 - gpt-4o-mini
 - text-embedding-ada-002

1. Deploy the gpt-4o model with those parameters

![image](/media/use-your-data/model_gpt4o.png)

2. Deploy the text embedding ada 002 model with those parameters

![image](/media/use-your-data/model_ada002.png)

## Add a Datasource to OpenAI

Before adding a data source in the OpenAI studio, you need to create a blob container in you storage account.

1. Go to your storage account and create a container named **documents**

2. Upload the sample files from the **documents** folder in this repository

2 in Azure AI Studio, add a Datasource un the **Chat** Section

![image](/media/use-your-data/aoai_chat.png)

3. Use **+Add a datasource**
   
4. Set the Datasource

    | Setting | Value |
    |---|---|
    | **Select Datasource** |  |
    | Type | **Azure Blob Storage (preview)** |
    | Select Azure Blob storage resource | **your storage account** |
    | Select Storage account container | **documents** |
    | Select Storage Azure AI Search | **your azure ai search resource** |
    | Enter the index name | **mydocuments** |
    | Indexer schedule | **Once** |
    | Add vector search to this search resource | **Checked** |
    | Indexer schedule | **Once** |
    | **Embedding model** |  |
    | Select and embedding model | **Azure OpenAI - text-embedding-ada-002** |

![image](/media/use-your-data/aoai_add_datasource.png)

5. Choose **Next** and leave as default for **Data management**

6. Choose **Next** and leave as default for **Data connection**

7. **Save and Create**

The indexing process will start. When it is finished, try to ask a quesiton about the documents in the Chat windows.

## Web app
The web app communicates with your Azure OpenAI resource. Since your Azure OpenAI resource has public network disabled, the web app needs to be set up to use the private endpoint in your virtual network to access your Azure OpenAI resource.

The web app needs to resolve your Azure OpenAI host name to the private IP of the private endpoint for Azure OpenAI. So, you need to configure the private DNS zone for your virtual network first.

1. [Create private DNS zone](/azure/dns/private-dns-getstarted-portal#create-a-private-dns-zone) in your resource group. 
1. [Add a DNS record](/azure/dns/private-dns-getstarted-portal#create-an-additional-dns-record). The IP is the private IP of the private endpoint for your Azure OpenAI resource, and you can get the IP address from the network interface associated with the private endpoint for your Azure OpenAI.
1. [Link the private DNS zone to your virtual network](/azure/dns/private-dns-getstarted-portal#link-the-virtual-network) so the web app integrated in this virtual network can use this private DNS zone.

When deploying the web app from Azure OpenAI Studio, select the same location with the virtual network, and select a proper SKU, so it can support the [virtual network integration feature](/azure/app-service/overview-vnet-integration). 

After the web app is deployed, from the Azure portal networking tab, configure the web app outbound traffic virtual network integration, choose the third subnet that you reserved for web app.

:::image type="content" source="../media/use-your-data/web-app-configure-outbound-traffic.png" alt-text="A screenshot showing outbound traffic configuration for the web app." lightbox="../media/use-your-data/web-app-configure-outbound-traffic.png":::

## Using the API

Make sure your sign-in credential has `Cognitive Services OpenAI Contributor` role on your Azure OpenAI resource, and run `az login` first.

:::image type="content" source="../media/use-your-data/api-local-test-setup-credential.png" alt-text="A screenshot showing the cognitive services OpenAI contributor role in the Azure portal." lightbox="../media/use-your-data/api-local-test-setup-credential.png":::

### Ingestion API

See the [ingestion API reference article](/rest/api/azureopenai/ingestion-jobs?context=/azure/ai-services/openai/context/context) for details on the request and response objects used by the ingestion API.

### Inference API

See the [inference API reference article](../references/on-your-data.md) for details on the request and response objects used by the inference API.
    
## Use Microsoft Defender for Cloud

You can now integrate [Microsoft Defender for Cloud](/azure/defender-for-cloud/defender-for-cloud-introduction) (preview) with your Azure resources to protect your applications. Microsoft Defender for Cloud protects your applications with [threat protection for AI workloads](/azure/defender-for-cloud/ai-threat-protection) , providing teams with evidence-based security alerts enriched with Microsoft threat intelligence signals and enables teams to strengthen their [security posture](/azure/defender-for-cloud/ai-security-posture) with integrated security best-practice recommendations.

Use [this form](https://forms.office.com/pages/responsepage.aspx?id=v4j5cvGGr0GRqy180BHbR9EXzLewuFRArQPJzR1tntlURThQR0hYU1MyRVRNODNMV1hBOUEzVlk3NC4u) to apply for access.
