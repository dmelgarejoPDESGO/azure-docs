---
title:  "Expose applications to the internet using Application Gateway with TLS termination"
titleSuffix: Azure Spring Cloud
description: How to expose applications to internet using Application Gateway with TLS termination
author: KarlErickson
ms.author: karler
ms.service: spring-cloud
ms.topic: how-to
ms.date: 11/09/2021
ms.custom: devx-track-java
---

# Expose applications to the internet with TLS Termination at Application Gateway

This article explains how to expose applications to the internet using Application Gateway.

When an Azure Spring Cloud service instance is deployed in your virtual network (VNET), applications on the service instance are only accessible in the private network. To make the applications accessible on the Internet, you need to integrate with Azure Application Gateway. The incoming encrypted traffic can be decrypted at the application gateway or it can be passed to Azure Spring Cloud encrypted to achieve end-to-end TLS/SSL. For dev and test purposes, you can start with SSL termination at the application gateway, which is covered in this guide. For production, we recommend end-to-end TLS/SSL with private certificate, as described in [Expose applications with end-to-end TLS in a virtual network](expose-apps-gateway-end-to-end-tls.md).

## Prerequisites

- [Azure CLI version 2.0.4 or later](/cli/azure/install-azure-cli).
- An Azure Spring Cloud service instance deployed in a virtual network with an application accessible over the private network using the default `.private.azuremicroservices.io` domain suffix. For more information, see [Deploy Azure Spring Cloud in a virtual network](how-to-deploy-in-azure-virtual-network.md)
- A custom domain to be used to access the application.
- A certificate, stored in Key Vault, which matches the custom domain to be used to establish the HTTPS listener. For more information, see [Tutorial: Import a certificate in Azure Key Vault](../key-vault/certificates/tutorial-import-certificate.md).

## Configure Application Gateway for Azure Spring Cloud

We recommend that the domain name, as seen by the browser, is the same as the host name which Application Gateway uses to direct traffic to the Azure Spring Cloud back end. This recommendation provides the best experience when using Application Gateway to expose applications hosted in Azure Spring Cloud and residing in a virtual network. If the domain exposed by Application Gateway is different from the domain accepted by Azure Spring Cloud, cookies and generated redirect URLs (for example) can be broken.

To configure Application Gateway in front of Azure Spring Cloud in a private VNET, use the following steps.

1. Follow the instructions in [Deploy Azure Spring Cloud in a virtual network](how-to-deploy-in-azure-virtual-network.md).
1. Follow the instructions in [Access your application in a private network](access-app-virtual-network.md).
1. Acquire a certificate for your domain of choice and store that in Key Vault. For more information, see [Tutorial: Import a certificate in Azure Key Vault](../key-vault/certificates/tutorial-import-certificate.md).
1. Configure a custom domain and corresponding certificate from Key Vault on an app deployed onto Azure Spring Cloud. For more information, see [Tutorial: Map an existing custom domain to Azure Spring Cloud](tutorial-custom-domain.md).
1. Deploy Application Gateway in a virtual network configured according to the following list:
   - Use Azure Spring Cloud in the backend pool, referenced by the domain suffixed with `private.azuremicroservices.io`.
   - Include an HTTPS listener using the same certificate from Key Vault.
   - Configure the virtual network with HTTP settings that use the custom domain name configured on Azure Spring Cloud instead of the domain suffixed with `private.azuremicroservices.io`.
1. Configure your public DNS to point to the application gateway.

## Define variables

Next, use the following commands to define variables for the resource group and virtual network you created as directed in [Deploy Azure Spring Cloud in a virtual network](how-to-deploy-in-azure-virtual-network.md). Replace the *\<...>* placeholders with real values based on your actual environment. When you define `SPRING_APP_PRIVATE_FQDN`, remove `https://` from the URI.

```bash
SUBSCRIPTION='<subscription-id>'
RESOURCE_GROUP='<resource-group-name>'
LOCATION='eastus'
SPRING_CLOUD_NAME='<name-of-azure-spring-cloud-instance>'
APPNAME='<name-of-app-in-azure-spring-cloud>'
SPRING_APP_PRIVATE_FQDN='$APPNAME.private.azuremicroservices.io'
VIRTUAL_NETWORK_NAME='azure-spring-cloud-vnet'
APPLICATION_GATEWAY_SUBNET_NAME='app-gw-subnet'
APPLICATION_GATEWAY_SUBNET_CIDR='10.1.2.0/24'
```

## Sign in to Azure

Use the following command to sign in to the Azure CLI and choose your active subscription.

```azurecli
az login
az account set --subscription $SUBSCRIPTION
```

## Configure the public domain name on Azure Spring Cloud

Traffic will enter the application deployed on Azure Spring Cloud using the public domain name. To configure your application to listen to this host name over HTTP, use the following commands to add a custom domain to your app, replacing the *\<...>* placeholders with real values:

```azurecli
KV_NAME='<name-of-key-vault>'
KV_RG='<resource-group-name-of-key-vault>'
CERT_NAME_IN_KV='<name-of-certificate-with-intermediaries-in-key-vault>'
DOMAIN_NAME=myapp.mydomain.com

az spring-cloud app custom-domain bind \
    --resource-group $RESOURCE_GROUP \
    --service $SPRING_CLOUD_NAME \
    --domain-name $DOMAIN_NAME \
    --app $APPNAME
```

## Create network resources

The application gateway to be created will join the same virtual network as the Azure Spring Cloud service instance. First, create a new subnet for the application gateway in the virtual network, then create a public IP address as the frontend of the application gateway, as shown in the following example.

```azurecli
APPLICATION_GATEWAY_PUBLIC_IP_NAME='app-gw-public-ip'
az network vnet subnet create \
    --name $APPLICATION_GATEWAY_SUBNET_NAME \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VIRTUAL_NETWORK_NAME \
    --address-prefix $APPLICATION_GATEWAY_SUBNET_CIDR
az network public-ip create \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --name $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
    --allocation-method Static \
    --sku Standard
```

### Create a managed identity for the application gateway

Your application gateway will need to be able to access Key Vault to read the certificate. To do this, the application gateway will use a user-assigned managed identity. For more information, see [What are managed identities for Azure resources?](../active-directory/managed-identities-azure-resources/overview.md). Create the managed identity by using the following command, replacing the *\<...>* placeholder:

```azurecli
APPGW_IDENTITY_NAME='<name-for-appgw-managed-identity>'
az identity create \
    --resource-group $RESOURCE_GROUP \
    --name $APPGW_IDENTITY_NAME
```

Then, use the following command to fetch the `objectId` for the managed identity. This value will be used later on to give rights to access the certificate in Key Vault.

```azurecli
APPGW_IDENTITY_CLIENTID=$(az identity show --resource-group $RESOURCE_GROUP --name $APPGW_IDENTITY_NAME --query clientId --output tsv)
APPGW_IDENTITY_OID=$(az ad sp show --id $APPGW_IDENTITY_CLIENTID --query objectId --output tsv)
```

### Set policy on Key Vault

Configure Key Vault using the following command so that the managed identity for the application gateway is allowed to access the certificate stored in Key Vault:

```azurecli
az keyvault set-policy \
    --resource-group $KV_RG \
    --name $KV_NAME \
    --object-id $APPGW_IDENTITY_OID \
    --secret-permissions get list \
    --certificate-permissions get list
```

## Create an application gateway

### [CLI](#tab/azure-cli)

Create an application gateway using `az network application-gateway create` and specify your application's private fully qualified domain name (FQDN) as servers in the backend pool. Be sure to use the user-assigned managed identity and point to the certificate in Key Vault using the certificate's secret ID.

```azurecli
APPGW_NAME='<name-for-application-gateway>'
CERT_NAME_IN_KV='<name-of-certificate-in-key-vault>'
KEYVAULT_SECRET_ID_FOR_CERT=$(az keyvault certificate show --name $CERT_NAME_IN_KV --vault-name $KV_NAME --query sid --output tsv)

az network application-gateway create \
    --name $APPGW_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --capacity 2 \
    --sku Standard_v2 \
    --frontend-port 443 \
    --http-settings-cookie-based-affinity Disabled \
    --http-settings-port 80 \
    --http-settings-protocol Http \
    --public-ip-address $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
    --vnet-name $VIRTUAL_NETWORK_NAME \
    --subnet $APPLICATION_GATEWAY_SUBNET_NAME \
    --servers $SPRING_APP_PRIVATE_FQDN \
    --key-vault-secret-id $KEYVAULT_SECRET_ID_FOR_CERT \
    --identity $APPGW_IDENTITY_NAME
```

It can take up to 30 minutes for Azure to create the application gateway.

### [Azure portal](#tab/azure-portal)

Create an application gateway using the following steps to enable SSL termination at the application gateway.

1. Sign in to the Azure portal and create a new Application Gateway resource.
1. Fill in the required fields for creating the application gateway. Leave the default values as they are.
1. After you provide a value for the **Virtual network** field, the **Subnet** field appears. Create a separate subnet for the application gateway in the VNET, as shown in the following screenshot.

   :::image type="content" source="media/expose-apps-gateway-tls-termination/create-application-gateway-basics.png" alt-text="Azure portal screenshot of 'Create application gateway' page.":::

1. Create a public IP address and assign it to the frontend of the application gateway, as shown in the following screenshot.

   :::image type="content" source="media/expose-apps-gateway-tls-termination/create-frontend-ip.png" alt-text="Azure portal screenshot showing Frontends tab of 'Create application gateway' page.":::

1. Create a backend pool for the application gateway. Select **Target** as your FQDN of the application deployed in Azure Spring Cloud.

   :::image type="content" source="media/expose-apps-gateway-tls-termination/create-backend-pool.png" alt-text="Azure portal screenshot of 'Add a backend pool' page.":::

1. Create a routing rule with HTTP listener.
   1. Select the public IP that you created earlier.
   1. Select **HTTPS** as protocol and **443** as port.
   1. Choose a certificate from Key Vault.
      1. Select the managed identity you created earlier.
      1. Select the right key vault and certificate, which were added to the key vault earlier.

         :::image type="content" source="media/expose-apps-gateway-tls-termination/create-routingrule-with-http-listener.png" alt-text="Azure portal screenshot of 'Add a routing rule' page.":::

   1. Select the **Backend targets** tab.

      :::image type="content" source="media/expose-apps-gateway-tls-termination/create-backend-http-settings.png" alt-text="Azure portal screenshot of 'Add a HTTP setting' page.":::

1. Select **Review and Create** to create the application gateway.

It can take up to 30 minutes for Azure to create the application gateway.

---

### Update HTTP settings to use the domain name towards the backend

Update the HTTP settings to use the public domain name as the hostname instead of the domain suffixed with `.private.azuremicroservices.io` to send traffic to Azure Spring Cloud with.

```azurecli
az network application-gateway http-settings update \
    --resource-group $RESOURCE_GROUP \
    --gateway-name $APPGW_NAME \
    --host-name-from-backend-pool false \
    --host-name $DOMAIN_NAME \
    --name appGatewayBackendHttpSettings
```

### Check the deployment of the application gateway

After it's created, check the backend health by using the following command. The output of this command enables you to determine whether the application gateway reaches your application through its private fully qualified domain name (FQDN).

```azurecli
az network application-gateway show-backend-health \
    --name $APPGW_NAME \
    --resource-group $RESOURCE_GROUP
```

The output indicates the healthy status of backend pool, as shown in the following example:

```output
{
  "backendAddressPools": [
    {
      "backendHttpSettingsCollection": [
        {
          "servers": [
            {
              "address": "my-azure-spring-cloud-hello-vnet.private.azuremicroservices.io",
              "health": "Healthy",
              "healthProbeLog": "Success. Received 200 status code",
              "ipConfiguration": null
            }
          ]
        }
      ]
    }
  ]
}
```

## Configure DNS and access the application

Configure the public DNS to point to the application gateway using a CNAME or A-record. You can find the public address for the application gateway by using the following command:

```azurecli
az network public-ip show \
    --resource-group $RESOURCE_GROUP \
    --name $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
    --query [ipAddress] \
    --output tsv
```

You can now access the application using the public domain name.

## Clean up resources

If you plan to continue working with subsequent articles, you might want to leave these resources in place. When no longer needed, delete the resource group, which deletes the resources in the resource group. To delete the resource group by using Azure CLI, use the following command:

```azurecli
az group delete --name $RESOURCE_GROUP
```

## Next steps

- [Exposing applications with end-to-end TLS in a virtual network](./expose-apps-gateway-end-to-end-tls.md)
- [Troubleshooting Azure Spring Cloud in VNET](./troubleshooting-vnet.md)
- [Customer Responsibilities for Running Azure Spring Cloud in VNET](./vnet-customer-responsibilities.md)
