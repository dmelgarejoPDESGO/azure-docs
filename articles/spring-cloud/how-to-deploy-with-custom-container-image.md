---
title: How to deploy applications in Azure Spring Cloud with a custom container image (Preview)
description: How to deploy applications in Azure Spring Cloud with a custom container image
author: karlerickson
ms.author: xiangy
ms.topic: how-to
ms.service: spring-cloud
ms.date: 4/28/2022
---

# Deploy an application with a custom container image (Preview)

**This article applies to:** ✔️ Standard tier ✔️ Enterprise tier

This article explains how to deploy Spring Boot applications in Azure Spring Cloud using a custom container image. Deploying an application with a custom container supports most features as when deploying a JAR application. Other Java and non-Java applications can also be deployed with the container image.

## Prerequisites

* A container image containing the application.
* The image is pushed to an image registry. For more information, see [Azure Container Registry](/azure/container-instances/container-instances-tutorial-prepare-acr).

> [!NOTE]
> The web application must listen on port `1025` for Standard tier and on port `8080` for Enterprise tier. The way to change the port depends on the framework of the application. For example, specify `SERVER_PORT=1025` for Spring Boot applications or `ASPNETCORE_URLS=http://+:1025/` for ASP.Net Core applications. The probe can be disabled for applications that do not listen on any port.

## Deploy your application

To deploy an application to a custom container image, use the following steps:

# [Azure CLI](#tab/azure-cli)

To deploy a container image, use one of the following commands:

* To deploy a container image to the public Docker Hub to an app, use the following command:

  ```azurecli
  az spring-cloud app deploy \
     --resource-group <your-resource-group> \
     --name <your-app-name> \
     --container-image <your-container-image> \
     --service <your-service-name>
  ```

* To deploy a container image from ACR to an app, or from another private registry to an app, use the following command:

  ```azurecli
  az spring-cloud app deploy \
     --resource-group <your-resource-group> \
     --name <your-app-name> \
     --container-image <your-container-image> \
     --service <your-service-name>
     --container-registry <your-container-registry> \
     --registry-password <your-password> |
     --registry-username <your-username>
   ```

To overwrite the entry point of the image, add the following two arguments to any of the above commands:

```azurecli
    --container-command "java" \
    --container-args "-jar /app.jar -Dkey=value"
```

To disable listening on a port for images that aren't web applications, add the following argument to the above commands:

```azurecli
    --disable-probe true
```

# [Portal](#tab/azure-portal)

1. Open the [Azure portal](https://portal.azure.com).
1. Open your existing Spring Cloud service instance.
1. Select **Apps** from left the menu, then select **Create App**.
1. Name your app, and in the **Runtime platform** pulldown list, select **Custom Container**.

   :::image type="content" source="media/how-to-deploy-with-custom-container-image/create-app-custom-container.png" alt-text="Azure portal screenshot of Create App page with Runtime platform dropdown showing and Custom Container selected." lightbox="media/how-to-deploy-with-custom-container-image/create-app-custom-container.png":::

1. Select **Edit** under *Image*, then fill in the fields as shown in the following image:

   :::image type="content" source="media/how-to-deploy-with-custom-container-image/custom-image-settings.png" alt-text="Azure portal screenshot showing the Custom Image Settings pane." lightbox="media/how-to-deploy-with-custom-container-image/custom-image-settings.png":::

   > [!NOTE]
   > The **Commands** and **Arguments** field are optional, which are used to overwrite the `cmd` and `entrypoint` of the image.
   >
   > You need to also specify the **Language Framework**, which is the web framework of the container image used. Currently, only **Spring Boot** is supported. For other Java applications or non-Java (polyglot) applications, select **Polyglot**.

1. Select **Save**, then select **Create** to deploy your application.

---

## Feature Support matrix

The following matrix shows what features are supported in each application type.

| Feature  | Spring Boot Apps - container deployment  | Polyglot Apps - container deployment  | Notes  |
|---|---|---|---|
| App lifecycle management                                        | ✔️ | ✔️ |   |
| Support for container registries                                | ✔️ | ✔️ |   |
| Assign endpoint                                                 | ✔️ | ✔️ |   |
| Azure Monitor                                                   | ✔️ | ✔️ |   |
| APM integration                                                 | ✔️ | ✔️ | Supported by [manual installation](#install-an-apm-into-the-image-manually)  |
| Blue/green deployment                                           | ✔️ | ✔️ |   |
| Custom domain                                                   | ✔️ | ✔️ |   |
| Scaling - auto scaling                                          | ✔️ | ✔️ |   |
| Scaling - manual scaling (in/out, up/down)                      | ✔️ | ✔️ |   |
| Managed Identity                                                | ✔️ | ✔️ |   |
| Spring Cloud Eureka & Config Server                             | ✔️ | ❌  |   |
| API portal for VMware Tanzu®                                    | ✔️ | ✔️ | Enterprise tier only  |
| Spring Cloud Gateway for VMware Tanzu®                          | ✔️ | ✔️ | Enterprise tier only  |
| Application Configuration Service for VMware Tanzu®             | ✔️ | ❌  | Enterprise tier only  |
| VMware Tanzu® Service Registry                                  | ✔️ | ❌  | Enterprise tier only  |
| VNET                                                            | ✔️ | ✔️ | Add registry to [allowlist in NSG or Azure Firewall](#avoid-not-being-able-to-connect-to-the-container-registry-in-a-vnet)  |
| Outgoing IP Address                                             | ✔️ | ✔️ |   |
| E2E TLS                                                         | ✔️ | ✔️ | Trust a self-signed CA is supported by [manual installation](#trust-a-certificate-authority-in-the-image)  |
| Liveness and readiness settings                                 | ✔️ | ✔️ |   |
| Advanced troubleshooting - thread/heap/JFR dump                 | ✔️ | ❌  | The image must include `bash` and JDK with `PATH` specified.   |
| Bring your own storage                                          | ✔️ | ✔️ |   |
| Integrate service binding with Resource Connector               | ✔️ | ❌  |   |
| Availability Zone                                               | ✔️ | ✔️ |   |
| App Lifecycle events                                            | ✔️ | ✔️ |   |
| Reduced app size - 0.5 vCPU and 512 MB                          | ✔️ | ✔️ |   |
| Automate app deployments with Terraform                         | ✔️ | ✔️ |   |
| Soft Deletion                                                   | ✔️ | ✔️ |   |
| Interactive diagnostic experience (AppLens-based)               | ✔️ | ✔️ |   |
| SLA                                                             | ✔️ | ✔️ |   |

> [!NOTE]
> Polyglot apps include non-Spring Boot Java, NodeJS, AngularJS, Python, and .NET apps.

## Common points to be aware of when deploying with a custom container

The following points will help you address common situations when deploying with a custom image.

### Trust a Certificate Authority in the image

To trust a CA in the image, set the following variables depending on your environment:

* You must import Java applications into the trust store by adding the following lines into your *Dockerfile*:

  ```dockerfile
  ADD EnterpriseRootCA.crt /opt/
  RUN keytool -keystore /etc/ssl/certs/java/cacerts -storepass changeit -noprompt -trustcacerts -importcert -alias EnterpriseRootCA -file /opt/EnterpriseRootCA.crt
  ```

* For Node.js applications, set the `NODE_EXTRA_CA_CERTS` environment variable:

  ```dockerfile
  ADD EnterpriseRootCA.crt /opt/
  ENV NODE_EXTRA_CA_CERTS="/opt/EnterpriseRootCA.crt"
  ```

* For Python, or other languages relying on the system CA store, on Debian or Ubuntu images, add the following environment variables:

  ```dockerfile
  ADD EnterpriseRootCA.crt /usr/local/share/ca-certificates/
  RUN /usr/sbin/update-ca-certificates
  ```

* For Python, or other languages relying on the system CA store, on CentOS or Fedora based images, add the following environment variables:

  ```dockerfile
  ADD EnterpriseRootCA.crt /etc/pki/ca-trust/source/anchors/
  RUN /usr/bin/update-ca-trust
  ```

### Avoid unexpected behavior when images change

When your application is restarted or scaled out, the latest image will always be pulled. If the image has been changed, the newly started application instances will use the new image while the old instances will continue to use the old image. Avoid using the `latest` tag or overwrite the image without a tag change to avoid unexpected application behavior.

### Avoid not being able to connect to the container registry in a VNet

If you deployed the instance to a VNet, make sure you allow the network traffic to your container registry in the NSG or Azure Firewall (if used). For more information, see [Customer responsibilities for running in VNet](/azure/spring-cloud/vnet-customer-responsibilities) to add the needed security rules.

### Install an APM into the image manually

The installation steps vary on different APMs and languages. The following steps are for New Relic with Java applications. You must modify the *Dockerfile* using the following steps:

1. Download and install the agent file into the image by adding the following to the *Dockerfile*:

   ```dockerfile
   ADD newrelic-agent.jar /opt/agents/newrelic/java/newrelic-agent.jar
   ```

1. Add the environment variables required by the APM:

   ```dockerfile
   ENV NEW_RELIC_APP_NAME=appName
   ENV NEW_RELIC_LICENSE_KEY=newRelicLicenseKey
   ```

1. Modify the image entry point by adding: `java -javaagent:/opt/agents/newrelic/java/newrelic-agent.jar`

To install the agents for other languages, refer to the official documentation for the other agents:

New Relic:

* Python: [Standard Python agent install](https://docs.newrelic.com/docs/apm/agents/python-agent/installation/standard-python-agent-install/)
* Node.js: [Install the Node.js agent](https://docs.newrelic.com/docs/apm/agents/nodejs-agent/installation-configuration/install-nodejs-agent/)

Dynatrace:

* Python: [Instrument Python applications with OpenTelemetry](https://www.dynatrace.com/support/help/extend-dynatrace/opentelemetry/opentelemetry-traces/opentelemetry-ingest/opent-python)
* Node.js: [Instrument Node.js applications with OpenTelemetry](https://www.dynatrace.com/support/help/extend-dynatrace/opentelemetry/opentelemetry-traces/opentelemetry-ingest/opent-nodejs)

AppDynamics:

* Python: [Install the Python Agent](https://docs.appdynamics.com/4.5.x/en/application-monitoring/install-app-server-agents/python-agent/install-the-python-agent)
* Node.js: [Installing the Node.js Agent](https://docs.appdynamics.com/4.5.x/en/application-monitoring/install-app-server-agents/node-js-agent/install-the-node-js-agent#InstalltheNode.jsAgent-install_nodejsInstallingtheNode.jsAgent)

### View the container logs

To view the console logs of your container application, the following CLI command can be used:

```azurecli
az spring-cloud app logs \
    --resource-group <your-resource-group> \
    --name <your-app-name> \
    --service <your-service-name> \
    --instance <your-instance-name>
```

To view the container events logs from the Azure Monitor, enter the query:

```query
AppPlatformContainerEventLogs
| where App == "hw-20220317-1b"
```

:::image type="content" source="media/how-to-deploy-with-custom-container-image/container-event-logs.png" alt-text="Screenshot of the container events log.":::

### Scan your image for vulnerabilities

We recommend that you use Microsoft Defender for Cloud with ACR to prevent your images from being vulnerable. For more information, see [Microsoft Defender for Cloud] (/azure/defender-for-cloud/defender-for-containers-introduction?tabs=defender-for-container-arch-aks#scanning-images-in-acr-registries)

### Switch between JAR deployment and container deployment

You can switch the deployment type directly by redeploying using the following command:

```azurecli
az spring-cloud app deploy \
    --resource-group <your-resource-group> \
    --name <your-app-name> \
    --container-image <your-container-image> \
    --service <your-service-name>
```

### Create another deployment with an existing JAR deployment

You can create another deployment using an existing JAR deployment using the following command:

```azurecli
az spring-cloud app deployment create \
    --resource-group <your-resource-group> \
    --name <your-deployment-name> \
    --app <your-app-name> \
    --container-image <your-container-image> \
    --service <your-service-name>
```

> [!NOTE]
> Automating deployments using Azure Pipelines Tasks or GitHub Actions are not currently supported.

## Next steps

* [How to capture dumps](/azure/spring-cloud/how-to-capture-dumps)
