# Secure DevOps and Secure AKS cluster

## Abstract

Aim of this repo is to prepare environment and process for secure development and secure operation of solution in DevOps environment for agile teams. Solution is build on top of AKS (kubernetes cluster) with istio service mesh extension and supporting container solution for specific Azure services (like KeyVault).

Lets try to deliver rules and boundaries for different roles in DevOps and organization operation teams.

Solution uses for demonstration set of microservices in different languages (AngularJS frontend in nginx, Java spring-boot, NodeJS) and utilizes different Azure services (AKS, Azure Container Registry, Azure Database for PostgreSQL, CosmosDB - with MongoDB API, KeyVault, Application Insights)

### Solution architecture

Picture describes architecture design of our microservice solution and also defines communication flows for components.

### Security constrains

* AKS is managed by AAD accounts (in cluster we can setup roles and grant access to users or security group)
* AKS installation and security setup is done by "admin" user, this type of authentication is later on not used for day-to-day tasks
* ACR (Azure Container Registry) also uses AAD authentication for users
* ACR authenticates AKS by service principal which is generated during AKS installation
* istio is used to setup mTLS communication in service mesh
* istio is used to control which services can communicate to which endpoints
* istio external service definition is used to control which services are used outside of cluster

### RBAC / authentication / Roles definition

For RBAC and authentication we will use kubernetes RBAC and direct connection for AKS to ADD where we can define users/groups and connect them with RBAC definitions for AKS.

* **AKS administrator** (the one who is able configure all AKS assets, setup istio rules for all namespaces)
* **Solution level** (namespace level)
    * **Security** (setting up credentials for DB and security rules for istio)
    * **Network** (define ingress/egress rules and routing)
    * **Deployment** (deploy solution)
    * **Operation** (see logs and component health - read permission to objects)

## Prerequisites 

Installed tools:
* kubectl - https://kubernetes.io/docs/tasks/tools/install-kubectl/
* helm - https://docs.helm.sh/using_helm/#installing-helm
* istio 1.0.2 - https://istio.io/docs/setup/kubernetes/download-release/
* az CLI - https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest

## Install necessary Azure Resources

All steps described in next captions expect that you have authenticated `az` CLI command line. For example you can use Azure Cloud Shell (https://shell.azure.com).

### Create Resource Group

Create resource group in your favorite region (check if AKS can be deployed there).

```bash
az group create --location northeurope --name AKSSEC
```

### Prepare AAD applications for AKS AAD authentication

Please follow these links and create Server and Client AAD applications. Collect necessary details, it means:
* Server APP ID
* Server App Secret
* Client App ID
* Tenant ID

https://docs.microsoft.com/en-us/azure/aks/aad-integration#create-server-application

https://docs.microsoft.com/en-us/azure/aks/aad-integration#create-client-application

### Deploy AKS cluster

Deploy cluster with RBAC and AAD enabled.

```bash
az aks create --resource-group AKSSEC --name akssec \
  --no-ssh-key --kubernetes-version 1.11.2 --node-count 2 --node-vm-size Standard_DS1_v2 \
  --aad-server-app-id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX   \
  --aad-server-app-secret '#############################'  \
  --aad-client-app-id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX \
  --aad-tenant-id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX  \
  --location northeurope
```

After successful deployment we will download kubernetes config file with "admin" credentials to finalize deployment and configuration. For day-to-day work will be used kubernetes config without admin private key and cluster will be authenticated via AAD authentication.

```bash
az aks get-credentials --name akssec --resource-group AKSSEC --admin
```

### Deploy Azure Container Registry (ACR)

Deploy container registry, please be aware that you have to provide unique name of ACR (int description you can see name `valdaakssec001` which has to be replaced with your registry name).

```bash
ACR_RESOURCE_GROUP=AKSSEC
ACR_NAME=valdaakssec001

az acr create --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --sku Standard --location northeurope
```

Grant access for AKS cluster to ACR registry with read permission.

```bash
AKS_RESOURCE_GROUP=AKSSEC
AKS_CLUSTER_NAME=akssec
ACR_RESOURCE_GROUP=AKSSEC
ACR_NAME=valdaakssec001

# Get the id of the service principal configured for AKS
CLIENT_ID=$(az aks show --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" --output tsv)

# Get the ACR registry resource id
ACR_ID=$(az acr show --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --query "id" --output tsv)

# Create role assignment
az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID

# Get ACR_URL for future use with docker-compose and build
export ACR_URL=$(az acr show --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --query "loginServer" --output tsv)
echo $ACR_URL
```

### Deploy Azure Database for PostgreSQL

First step is to create PostgreSQL database. Be aware that name of PostgreSQL server has to be unique, you have to change name `valdaakspostgresql001` to some unique name fits to your setup.

```bash
# create PostgreSQL server in Azure
az postgres server create --resource-group AKSSEC \
  --name valdaakspostgresql001  --location northeurope \
  --admin-user myadmin --admin-password VerySecurePassword123... \
  --sku-name B_Gen5_1 --version 9.6

# Get PostgreSQL FQDN (we will need in later on for configuration)
POSTGRES_FQDN=$(az postgres server show --resource-group AKSSEC --name valdaakspostgresql001 --query "fullyQualifiedDomainName" --output tsv)
echo $POSTGRES_FQDN
```
After successful deployment we will create regular database `todo` for our microservice. 

```bash
# create PostgreSQL database in Azure
az postgres db create --resource-group AKSSEC \
  --server-name valdaakspostgresql001   \
  --name todo
```

*Note: In real scenario please create also non-privileged DB user for access your databases on PostgreSQL server!*

### Deploy CosmosDB with Mongo API

Now lets create CosmosDB instance with MongoDB API for our solution. Because database uses global FQDN please replace name `valdaaksmongodb001` by name which fits your needs.

```bash
# Create CosmosDB instance
az cosmosdb create --name valdaaksmongodb001 --kind MongoDB --resource-group AKSSEC
```

### Deploy Application insights

Application insights are used for collecting application logs, diagnostic and traces.

```bash
# Create Application Insights
az resource create -g AKSSEC -n akssecappinsights001 --resource-type microsoft.insights/components --api-version '2015-05-01' --is-full-object --properties '{
  "location": "northeurope",
  "tags": {},
  "kind": "web",
  "properties": {
    "Application_Type": "web",
    "Flow_Type": "Bluefield",
    "Request_Source": "rest"
  }
}'

# Collect Instrumentation key - we will need it later on
APPINSIGHT_KEY=$(APPINSIGHT_KEY= az resource show -g AKSSEC -n akssecappinsights001 --resource-type microsoft.insights/components --query "properties.InstrumentationKey" -o tsv)
echo $APPINSIGHT_KEY
```

### Create KeyVault for storing security assets

### Build images and push to Azure Container Registry

Images for solution are build via one docker-compose file, in real situation there will be some separated CI?CD pipeline for each microservice to achieve individual lifecycle.

## Appendix

Source codes of all applications are stored in folder /src, whole solution is build by docker-compose.

Also integration with CI/CD pipeline is not in scope of this document, because preparation of AKS cluster (include RBAC, istio configuration) has to be done first. Microservice deployment is done from command line by helm tool, helm can be integrated easily with any CI/CD pipeline.

