# Secure DevOps and Secure AKS cluster

*created by Valdemar Zavadsky (Microsoft)*

*consulted by Jaroslav Seps (ESET)*

## Abstract

Aim of this repo is to prepare environment and process for secure development and secure operation of solution in DevOps environment for agile teams. Solution is build on top of AKS (kubernetes cluster) with istio service mesh extension and supporting container solution for specific Azure services (like KeyVault).

Lets try to deliver rules and boundaries for different roles in DevOps and organization operation teams.

Solution uses for demonstration set of microservices in different languages (AngularJS frontend in nginx, Java spring-boot, NodeJS) and utilizes different Azure services (AKS, Azure Container Registry, Azure Database for PostgreSQL, CosmosDB - with MongoDB API, KeyVault, Application Insights)

### Solution architecture

Picture describes architecture design of our microservice solution and also defines communication flows for components. In scope of our design are internal components of AKS and deployed solutions, especially:

* cluster deployment
* role definition for RBAC
* inter-service communication
* end point definition
* secrets handling
* private container registry

What is not in scope:

* real internet perimeter (WAF, Application Gateway, HTTPS offloading)
* VNET perimeter, there is no Firewall, micro-segmentation rules
* there is no rules for AKS administration, in real world we will need in this area more rules which are supporting DevOps team. In fact we can use one AKS admin role which can handle all task or we separate responsibilities to more roles like:
  * Global cluster administrator
  * Istio administrator
  * RBAC administrator
  * Storage classes administrator

Application architecture (also see in picture):

* SPA - single page angular application which is hosted in nginx HTML pages
* ToDo - java-spring-boot application which stores ToDo items in PostgreSQL database, API are exposed on /api/todo endpoint.
* Log - internal microservice which collects data manipulation operations in services(in our case in ToDo service). It is called only from ToDo microservice.

![arcitecture schema](./img/aks.png)

### Security constrains

* AKS is managed by AAD accounts (in cluster we can setup roles and grant access to users or security group)
* AKS installation and security setup is done by "admin" user, this type of authentication is later on not used for day-to-day tasks
* ACR (Azure Container Registry) also uses AAD authentication for users
* ACR authenticates AKS by service principal which is generated during AKS installation
* istio is used to setup mTLS communication in service mesh
* istio is used to control which services can communicate to which endpoints
* istio external service definition is used to control which services are used outside of cluster

### RBAC / authentication / Roles definition

For RBAC and authentication we will use kubernetes RBAC and direct connection for AKS to ADD where we can define users/groups and connect them with RBAC definitions for AKS. These rules are targeted like example for one DevOps team working with AKS cluster, you can use more independent DevOps teams and define different rules (more rules with specific rights or rules with integrated responsibility). These DevOps rules are managed by cluster admin or cluster admin rules.

* **AKS administrator** (the one who is able configure all AKS assets, setup istio rules for all namespaces)
* **Solution level** (namespace level)
    * **Security** (setting up credentials for DB and security rules for istio)
    * **Mesh** (define ingress/egress rules, routing and service mesh rules)
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
export AKS_RESOURCE_GROUP=AKSSEC
export LOCATION="northeurope"

az group create --location ${LOCATION} --name ${AKS_RESOURCE_GROUP}
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
export AKS_RESOURCE_GROUP=AKSSEC
export LOCATION="northeurope"
export AKS_CLUSTER_NAME=akssec

az aks create --resource-group ${AKS_RESOURCE_GROUP} --name ${AKS_CLUSTER_NAME} \
  --no-ssh-key --kubernetes-version 1.11.3 --node-count 3 --node-vm-size Standard_DS1_v2 \
  --aad-server-app-id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX   \
  --aad-server-app-secret '#############################'  \
  --aad-client-app-id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX \
  --aad-tenant-id XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX  \
  --location ${LOCATION}
```

After successful deployment we will download kubernetes config file with "admin" credentials to finalize deployment and configuration. For day-to-day work will be used kubernetes config without admin private key and cluster will be authenticated via AAD authentication.

```bash
az aks get-credentials --name ${AKS_CLUSTER_NAME} --resource-group ${AKS_RESOURCE_GROUP} --admin
```

### Deploy Azure Container Registry (ACR)

Deploy container registry, please be aware that you have to provide unique name of ACR (int description you can see name `valdaakssec001` which has to be replaced with your registry name).

```bash
ACR_RESOURCE_GROUP=AKSSEC
export LOCATION="northeurope"
ACR_NAME=valdaakssec001

az acr create --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --sku Standard --location ${LOCATION}
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
export POSTGRESQL_NAME="valdaakspostgresql001"
export POSTGRESQL_RG="AKSSEC"
# create PostgreSQL server in Azure
az postgres server create --resource-group ${POSTGRESQL_RG} \
  --name ${POSTGRESQL_NAME}  --location ${LOCATION} \
  --admin-user myadmin --admin-password VerySecurePassword123... \
  --sku-name B_Gen5_1 --version 9.6

# Get PostgreSQL FQDN (we will need in later on for configuration)
POSTGRES_FQDN=$(az postgres server show --resource-group ${POSTGRESQL_RG} --name ${POSTGRESQL_NAME} --query "fullyQualifiedDomainName" --output tsv)
echo $POSTGRES_FQDN
```

After successful deployment we will create regular database `todo` for our microservice. 

```bash
# create PostgreSQL database in Azure
az postgres db create --resource-group ${POSTGRESQL_RG} \
  --server-name ${POSTGRESQL_NAME}   \
  --name todo

# enable access for Azure resources (only for services running in azure)
az postgres server firewall-rule create \
  --server-name ${POSTGRESQL_NAME} \
  --resource-group ${POSTGRESQL_RG} \
  --name "AllowAllWindowsAzureIps" --start-ip-address "0.0.0.0" --end-ip-address "0.0.0.0"
```

*Note: In real scenario please create also non-privileged DB user for access your databases on PostgreSQL server! Also we can use Service Endpoint to establish connection from VNET to PostgreSQL for more robust security.*

### Deploy CosmosDB with Mongo API

Now lets create CosmosDB instance with MongoDB API for our solution. Because database uses global FQDN please replace name `valdaaksmongodb001` by name which fits your needs.

```bash
export COSMOSDB_NAME="valdaaksmongodb001"
export COSMOSDB_RG=AKSSEC
# Create CosmosDB instance
az cosmosdb create --name ${COSMOSDB_NAME} --kind MongoDB --resource-group ${COSMOSDB_RG}
```

### Deploy Application insights

Application insights are used for collecting application logs, diagnostic and traces. Later on you can check how your application works, how many requests were called and also App Insights can collect application logs for later analyses.

```bash
# Create Application Insights
export APPINSIGHTS_RG=AKSSEC
export APPINSIGTHS_NAME="akssecappinsights001"
az resource create -g ${APPINSIGHTS_RG} -n ${APPINSIGTHS_NAME} --resource-type microsoft.insights/components --api-version '2015-05-01' --is-full-object --properties '{
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
APPINSIGHT_KEY=$(APPINSIGHT_KEY= az resource show -g ${APPINSIGHTS_RG} -n ${APPINSIGTHS_NAME} --resource-type microsoft.insights/components --query "properties.InstrumentationKey" -o tsv)
echo $APPINSIGHT_KEY
```

### Create KeyVault for storing security assets (now optional)

In this step we will create KeyVault for storing keys and secrets, KeyVault name has to be unique in Azure, please change name `valdaakskeyvault001` which will fit your needs.

```bash
export KEYVAULT_NAME="valdaakskeyvault001"
export KEYVAULT_RG="AKSSEC"

# create keyvault
az keyvault create -n ${KEYVAULT_NAME} -g ${KEYVAULT_RG}

# setup access to AKS
CLIENT_ID=$(az aks show --resource-group ${AKS_RESOURCE_GROUP} --name ${AKS_CLUSTER_NAME} --query "servicePrincipalProfile.clientId" --output tsv)
az keyvault set-policy -n ${KEYVAULT_NAME} \
  --key-permissions get list decrypt \
  --secret-permissions get list \
  --certificate-permissions get list \
  --spn $CLIENT_ID

# create one testing secret
az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name 'TEST' --value 'TEST-PWD'
```

## Initialize kubernetes cluster

Now we can download kubernetes credentials for admin and install helm.

```bash
# download kubernetes config for admin
az aks get-credentials --name ${AKS_CLUSTER_NAME} --resource-group ${AKS_RESOURCE_GROUP} --admin

# patch kubernetes configuration to be able to access control plane
kubectl create clusterrolebinding kubernetes-dashboard \
  -n kube-system --clusterrole=cluster-admin \
  --serviceaccount=kube-system:kubernetes-dashboard
```

## Test current status of deployment (now unsecured)

### Access kubernetes by admin account

Now we can download kubernetes credentials for admin and try to access cluster control plane.

```bash
# download kubernetes config for admin
az aks get-credentials --name ${AKS_CLUSTER_NAME} --resource-group ${AKS_RESOURCE_GROUP} --admin

# run proxy
kubectl proxy
```

Now we can access kubernetes control plane on address http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy/

## Initialize helm

```bash
# Create a service account for Helm and grant the cluster admin role.
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
EOF

# initialize helm
helm init --service-account tiller --upgrade

# after while check if helm is installed in cluster
helm version
```

## Build images and push to Azure Container Registry

Images for solution are build via one docker-compose file, in real situation there will be some separated CI/CD pipeline for each microservice to achieve individual lifecycle.

### Login to ACR with AAD (user)

You have to change name `valdaakssec001` to name of registry which you created.

```bash
# name of registry
export ACR_NAME=valdaakssec001

# login to registry
az acr login --name $ACR_NAME
```

### Build and push images

First step is to clone sources from github to your machine `git clone git@github.com:valda-z/aks-secure-devops.git`.
Than enter directory with sources, sources are located in directory `./src`.

Please check that you have initialized variable `ACR_URL` with full name of your ACR registry (see description above how to read ACR_URL) or you can set variable and get value from Azure portal.

```bash
# Get ACR_URL for future use with docker-compose and build
export ACR_URL=$(az acr show --name $ACR_NAME --resource-group ${ACR_RESOURCE_GROUP} --query "loginServer" --output tsv)
echo $ACR_URL

# enter to src directory
cd src

# build images and push to registry
docker-compose build
docker-compose push
```

### Create namespace for our test

```bash
kubectl create namespace mytest
```

### Install nging ingress controller

This command will create nginx ingress controler which will be used for communication from external world via public IP.

```bash
# install nginx
helm install --name default-ingress stable/nginx-ingress --namespace mytest

# wait for deployment - we have to collect public IP address.
kubectl get svc --namespace mytest

```

Command `kubectl get svc --namespace mytest` has to return External IP for ingress controller, in my case it is `23.101.56.104`, we will need this address in future for configuring DNS rules.
In real world you will put this IP to your DNS registry to point some DNS name to this address.

```text
NAME                                            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
default-ingress-nginx-ingress-controller        LoadBalancer   10.0.89.222    23.101.56.104   80:31661/TCP,443:32461/TCP   2m
default-ingress-nginx-ingress-default-backend   ClusterIP      10.0.217.153   <none>          80/TCP                       2m
```

### Deploy our microservices via helm to namespace `mytest`

It will be deployed with release name `myreleasespa`, `myreleaselog` and `myreleasetodo`, these names are also part of secrets definitions.

#### Create secrets - it will contain connection string information

```bash
# Collect Instrumentation key
APPINSIGHT_KEY=$(APPINSIGHT_KEY= az resource show -g ${APPINSIGHTS_RG} -n ${APPINSIGHTS_NAME} --resource-type microsoft.insights/components --query "properties.InstrumentationKey" -o tsv)
echo $APPINSIGHT_KEY

# create secrets for myappspa
kubectl create secret generic myreleasespa-myappspa \
  --from-literal=appinsightskey="$APPINSIGHT_KEY" \
  --namespace mytest

# create secrets for myapptodo
POSTGRESQL_NAME=valdaakspostgresql001
POSTGRESQL_USER=myadmin
POSTGRESQL_PASSWORD=VerySecurePassword123...
POSTGRESQL_URL="jdbc:postgresql://${POSTGRESQL_NAME}.postgres.database.azure.com:5432/todo?user=${POSTGRESQL_USER}@${POSTGRESQL_NAME}&password=${POSTGRESQL_PASSWORD}&ssl=true"
kubectl create secret generic myreleasetodo-myapptodo \
  --from-literal=appinsightskey="$APPINSIGHT_KEY" \
  --from-literal=postgresqlurl="$POSTGRESQL_URL" \
  --namespace mytest

# create secrets for myapplog
MONGO_DB="${COSMOSDB_NAME}"
MONGO_PWD=$(az cosmosdb list-keys --name $MONGO_DB --resource-group ${COSMOSDB_RG}  --query "primaryMasterKey" --output tsv)
kubectl create secret generic myreleaselog-myapplog \
  --from-literal=appinsightskey="$APPINSIGHT_KEY" \
  --from-literal=mongodb="$MONGO_DB" \
  --from-literal=mongopwd="$MONGO_PWD" \
  --namespace mytest
```

#### Deploy micro-services

Lets use DNS name created dynamically from IP Address - in my case `23.101.56.104.xip.io`.

##### myappspa

Microservice with frontend (html5/angularjs/nginx) which is exposed to internet via nginx ingress controller.
Change ACR name `valdaakssec001` to your name of ACR.

```bash
# run helm installation
helm upgrade --install myreleasespa myappspa --set-string image.repository='valdaakssec001.azurecr.io/myappspa',image.tag='1',ingress.host='23.101.56.104.xip.io' --namespace='mytest'
```

##### myapplog

Microservice with backend (nodejs/mongo) which is exposed only internaly to cluster.
Change ACR name `valdaakssec001` to your name of ACR.

```bash
# run helm installation
helm upgrade --install myreleaselog myapplog --set-string image.repository='valdaakssec001.azurecr.io/myapplog',image.tag='1' --namespace='mytest'
```

##### myapptodo

Microservice with backend (java/postgres) which is exposed to internet via nginx ingress controller to path /api/todo.
Change ACR name `valdaakssec001` to your name of ACR.

```bash
# run helm installation
helm upgrade --install myreleasetodo myapptodo --set-string image.repository='valdaakssec001.azurecr.io/myapptodo',image.tag='1',logservice.url='http://myreleaselog-myapplog:8080/api/log',ingress.host='23.101.56.104.xip.io' --namespace='mytest'
```

### Test our application deployment

Now we can access application UI on URL http://23.101.56.104.xip.io (IP address depends on your configuration).

#### Cleanup cluster after test

Cleanup our deployment, services and secrets.

```bash
# delete microservice deployments
helm delete myreleasespa --purge
helm delete myreleaselog --purge
helm delete myreleasetodo --purge

# delete nginx deployment
helm delete default-ingress --purge

# delete namespace
kubectl delete namespace mytest
```

## Cleanup helm tiller

Because helm tiller run under service account we cannot segregate roles for namespace, in fact tiller is namespace admin and event if user has limited access to namespace via helm he can do any tasks in namespace.
This is reason why we will use different approach with helm and istio in our secure AKS cluster.

```bash
# delete role for helm tiller
kubectl delete ClusterRoleBinding tiller

# delete tiller system account
kubectl delete ServiceAccount tiller --namespace kube-system

# remove helm  tiller from cluster
helm reset --force

# check that tiller is gone
kubectl get pods --all-namespaces
```

## Setup istio

Istio description can be found there: https://istio.io . There we can also download last version of istio, this guide is based on version `1.0.2`.
Please download this version and follow installation guide on istio site: https://istio.io/docs/setup/kubernetes/download-release/ 

### architecture with istio

On following picture we can se ehow istio handles communication in cluster, deep explanation of istio function can be found on istio website (https://istio.io/docs/concepts/security/).

Istio high-level architecture overview:

![istio arch](https://istio.io/docs/concepts/security/architecture.svg)

Changes in our microservice solution are displayed on following picture, changes can be described like:

* communication between PODs are done via istio proxy
* istio proxy enforces mTLS protokol
* istio enforces communication rules (micro-service to micro-service communication rules)
* enforcing egress communication for PODs - micro-services can communicate only to other services handled by istio, for communication with external world we have to create virtual service which creates envelope arround services from outside of istio.

![our app](img/arch.png)

### install instio 

Described steps expects that you are in working folder with istio installation on your machine `~/istio-1.0.2`. Istio is installed via kubectl commandline utility, process expects that we are logged in via admin kubernetes config file.

```bash
# install istio via helm to cluster
# - prepare installation script
helm template install/kubernetes/helm/istio --name istio --namespace istio-system --set sidecarInjectorWebhook.enabled=true,mtls.enabled=true > $HOME/istio.yaml
# - crate namespce for istio
kubectl create namespace istio-system
# - install istio
kubectl apply -f $HOME/istio.yaml

# check istio components installation status (and wait for success deployment)
kubectl get pods --namespace istio-system -w
```

## Setup namespace and roles for RBAC

### Create namespace for our experiment

```bash
# create namespace
kubectl create namespace istiotest
```

### Create roles

Roles are created by user with admin rights in AKS cluster.
Role bindings are configured with user accounts, in real case wecan use bindings to security groups from AAD.

```bash
# create reader group for istiotest
cat <<EOF | kubectl apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: istiotest
  name: reader-istiotest
rules:
- apiGroups:
  - ""
  - "extensions"
  resources:
    - pods
    - events
    - services
    - ingress
    - deployments
    - configmaps
    - replicasets
    - replicationcontrollers
  verbs:
    - get
    - watch
    - list
EOF

cat <<EOF | kubectl apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: istiotest
  name: deployment-istiotest
rules:
- apiGroups:
  - ""
  - "extensions"
  resources:
    - pods
    - events
    - services
    - ingress
    - deployments
    - configmaps
    - replicasets
    - replicationcontrollers
  verbs:
    - get
    - watch
    - list
- apiGroups:
  - ""
  - "extensions"
  resources:
    - pods
    - pods/exec
    - deployments
    - configmaps
    - replicasets
    - replicationcontrollers
  verbs:
    - get
    - watch
    - list
    - create
    - update
    - patch
    - delete
    - deletecollection
    - exec
EOF

cat <<EOF | kubectl apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: istiotest
  name: mesh-istiotest
rules:
- apiGroups:
  - ""
  - "extensions"
  resources:
    - pods
    - events
    - services
    - ingress
    - deployments
    - configmaps
    - replicasets
    - replicationcontrollers
  verbs:
    - get
    - watch
    - list
- apiGroups:
  - ""
  - "extensions"
  resources:
    - services
  verbs:
    - get
    - watch
    - list
    - create
    - update
    - patch
    - delete
    - deletecollection
- apiGroups:
  - "networking.istio.io"
  resources:
    - virtualservices
    - serviceentries
    - envoyfilters
    - destinationrules
    - gateways
  verbs:
    - get
    - watch
    - list
    - create
    - update
    - patch
    - delete
    - deletecollection
- apiGroups:
  - "config.istio.io"
  resources:
    - bypasses
    - attributemanifests
    - deniers
    - reportnothings
    - listcheckers
    - handlers
    - instances
    - rules
    - listentries
    - checknothings
    - noops
  verbs:
    - get
    - watch
    - list
    - create
    - update
    - patch
    - delete
    - deletecollection
EOF

cat <<EOF | kubectl apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: istiotest
  name: secret-istiotest
rules:
- apiGroups:
  - ""
  resources:
    - secrets
  verbs:
    - get
    - watch
    - list
    - create
    - update
    - patch
    - delete
    - deletecollection
EOF

cat <<EOF | kubectl create -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: usr-reader-istiotest
  namespace: istiotest
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: "aks.reader@valda.cloud"
roleRef:
  kind: Role
  name: reader-istiotest
  apiGroup: rbac.authorization.k8s.io
EOF

cat <<EOF | kubectl create -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: usr-deployment-istiotest
  namespace: istiotest
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: "aks.deployment@valda.cloud"
roleRef:
  kind: Role
  name: deployment-istiotest
  apiGroup: rbac.authorization.k8s.io
EOF

cat <<EOF | kubectl create -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: usr-secret-istiotest
  namespace: istiotest
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: "aks.secret@valda.cloud"
roleRef:
  kind: Role
  name: secret-istiotest
  apiGroup: rbac.authorization.k8s.io
EOF

cat <<EOF | kubectl create -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: usr-mesh-istiotest
  namespace: istiotest
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: "aks.mesh@valda.cloud"
roleRef:
  kind: Role
  name: mesh-istiotest
  apiGroup: rbac.authorization.k8s.io
EOF

```

## Deploy microservice app with istio

We will use automatic sidecar injection for new namespace, there is no need to involve any istio utilities to deployment pipeline. All istio configurations with TLS, routes, etc. are done separately by different cluster roles.

### Configure namespace with sidecar injection

Automatic sidecar injection enforces that deployment of PODs will be decorated on fly with istio proxy and rules and will be connected to istio subsystem.

```bash
# check if sidecar injection is installed
# expected result is:
#   admissionregistration.k8s.io/v1alpha1
#   admissionregistration.k8s.io/v1beta1
kubectl api-versions | grep admissionregistration

# enable sidecar injection on namespace
kubectl label namespace istiotest istio-injection=enabled

# check if sidecar injection is enabled
kubectl get namespace -L istio-injection
```

### Create secrets - it will contain connection string information

Operations have to be done with user in role `secret-istiotest`.

```bash
# Collect Instrumentation key
APPINSIGHT_KEY=$(APPINSIGHT_KEY= az resource show -g ${APPINSIGHTS_RG} -n ${APPINSIGHTS_NAME} --resource-type microsoft.insights/components --query "properties.InstrumentationKey" -o tsv)
echo $APPINSIGHT_KEY

# create secrets for myappspa
kubectl create secret generic myreleasespa-myappspa \
  --from-literal=appinsightskey="$APPINSIGHT_KEY" \
  --namespace istiotest

# create secrets for myapptodo
POSTGRESQL_NAME=valdaakspostgresql001
POSTGRESQL_USER=myadmin
POSTGRESQL_PASSWORD=VerySecurePassword123...
POSTGRESQL_URL="jdbc:postgresql://${POSTGRESQL_NAME}.postgres.database.azure.com:5432/todo?user=${POSTGRESQL_USER}@${POSTGRESQL_NAME}&password=${POSTGRESQL_PASSWORD}&ssl=true"
kubectl create secret generic myreleasetodo-myapptodo \
  --from-literal=appinsightskey="$APPINSIGHT_KEY" \
  --from-literal=postgresqlurl="$POSTGRESQL_URL" \
  --namespace istiotest

# create secrets for myapplog
MONGO_DB="${COSMOSDB_NAME}"
MONGO_PWD=$(az cosmosdb list-keys --name $MONGO_DB --resource-group ${COSMOSDB_RG}  --query "primaryMasterKey" --output tsv)
kubectl create secret generic myreleaselog-myapplog \
  --from-literal=appinsightskey="$APPINSIGHT_KEY" \
  --from-literal=mongodb="$MONGO_DB" \
  --from-literal=mongopwd="$MONGO_PWD" \
  --namespace istiotest
```

### Deploy microservices via helm to `istiotest` namespace

```bash
# myapplog - create deployment yaml
helm template myapplog --name myreleaselog --namespace='istiotest' --set-string image.repository='valdaakssec001.azurecr.io/myapplog',image.tag='1',template.deployment=true > myapplog-deployment.yaml

# run myapplog deployment - user has to be in group deployment-istiotest
kubectl apply -f myapplog-deployment.yaml --namespace istiotest

# myappspa - create deployment yaml
helm template myappspa --name myreleasespa --namespace='istiotest' --set-string image.repository='valdaakssec001.azurecr.io/myappspa',image.tag='1',template.deployment=true > myappspa-deployment.yaml

# run myappspa deployment - user has to be in group deployment-istiotest
kubectl apply -f myappspa-deployment.yaml --namespace istiotest

# myapptodo - create deployment yaml
helm template myapptodo --name myreleasetodo --namespace='istiotest' --set-string image.repository='valdaakssec001.azurecr.io/myapptodo',logservice.url='http://myreleaselog-myapplog:8080/api/log',image.tag='1',template.deployment=true > myapptodo-deployment.yaml

# run myapptodo deployment - user has to be in group deployment-istiotest
kubectl apply -f myapptodo-deployment.yaml --namespace istiotest
```

### Deploy microservices network via helm to `istiotest` namespace

Change name of ACR registry from `valdaakssec001` to your name used during deployment. Same with MongoDB name `valdaaksmongodb001` and PostgreSQL name `valdaakspostgresql001`.

```bash
# myapplog - create deployment yaml
helm template myapplog --name myreleaselog --namespace='istiotest' --set-string image.repository='valdaakssec001.azurecr.io/myapplog',image.tag='1',database.mongodb='valdaaksmongodb001.documents.azure.com',template.service=true > myapplog-service.yaml

# run myapplog deployment - user has to be in group deployment-istiotest
kubectl apply -f myapplog-service.yaml --namespace istiotest

# myappspa - create deployment yaml
helm template myappspa --name myreleasespa --namespace='istiotest' --set-string image.repository='valdaakssec001.azurecr.io/myappspa',image.tag='1',template.service=true > myappspa-service.yaml

# run myappspa deployment - user has to be in group deployment-istiotest
kubectl apply -f myappspa-service.yaml --namespace istiotest

# myapptodo - create deployment yaml
helm template myapptodo --name myreleasetodo --namespace='istiotest' --set-string image.repository='valdaakssec001.azurecr.io/myapptodo',image.tag='1',database.postgres='valdaakspostgresql001.postgres.database.azure.com',template.service=true > myapptodo-service.yaml

# run myapptodo deployment - user has to be in group deployment-istiotest
kubectl apply -f myapptodo-service.yaml --namespace istiotest
```

### expose microservices - gateway

```bash
# create gateway and routes to service
cat <<EOF | kubectl create --namespace istiotest -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: myapp-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-gateway-myappspa
spec:
  hosts:
  - "*"
  gateways:
  - myapp-gateway
  http:
  - route:
    - destination:
        host: myreleasespa-myappspa
        port:
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-gateway-myapptodo
spec:
  hosts:
  - "*"
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /api/todo
    route:
    - destination:
        host: myreleasetodo-myapptodo
        port:
          number: 80
EOF
```

### get istio gateway IP

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Now we can test our web solution on `http://IP-ADDRESS`

### call services in mesh

First we need some pod which can be used for testing and hacking in cluster.

```bash
# create pod with busybox
cat <<EOF | kubectl apply --namespace istiotest -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: client-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
        - name: client
          image: tutum/curl
          command: ["tail"]
          args: ["-f", "/dev/null"]
EOF

# get client POD name (for client-deployment-####)
kubectl get pods --namespace istiotest

# connect to pod
kubectl exec --namespace istiotest -it client-deployment-846b476b99-nnbrx  -- /bin/bash

### in container bash
  # test access external service - has to fail
  curl https://github.com 
  # test internal services - has to work
  curl -v http://myreleaselog-myapplog:8080
  curl -v http://myreleasetodo-myapptodo:80
```

### Create deny rule for myapplog microservice

This rule will deny all requests except requests from service myapptodo to myapplog.

```bash
cat <<EOF | kubectl apply --namespace istiotest -f -
apiVersion: "config.istio.io/v1alpha2"
kind: denier
metadata:
  name: deny-myapplog-handler
spec:
  status:
    code: 7
    message: Not allowed
---
apiVersion: "config.istio.io/v1alpha2"
kind: checknothing
metadata:
  name: deny-myapplog-request
spec:
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: deny-myapplog-rule
spec:
  match: destination.labels["app"] == "myapplog" && source.labels["app"] != "myapptodo"
  actions:
  - handler: deny-myapplog-handler.denier
    instances: [ deny-myapplog-request.checknothing ]
EOF
```

### check service communication with denial rule on `myapplog` service

```bash
# connect to pod
kubectl exec --namespace istiotest -it client-deployment-846b476b99-nnbrx  -- /bin/bash

### in container bash
  # test access external service - has to fail
  curl https://github.com 
  # test internal services
    # myapplog has to fail
    curl -v http://myreleaselog-myapplog:8080
    # myapptodo has to work
    curl -v http://myreleasetodo-myapptodo:80
```

## AKS and secrets

In current description we are using kubernetes secrets for storing sensitive data like connection strings. These secrets are stored via KMS kubernetes service in master nodes - encryption is done by internal rules of kubernetes nodes. 
In future versions of AKS we will be able to use Azure KeyVault for encrypting secrets handled by KMS service - see this link: https://github.com/Azure/kubernetes-kms#-acs-engine - in fact we can test now this functionality only in acs-enginee.

Or we can use different approach with kubernetes-keyvault-flexvolumes authenticated by service principal or POD identities.
https://github.com/Azure/kubernetes-keyvault-flexvol#option-2---aks-azure-kubernetes-service-manually

*TODO: we will create description of experiment later...*

## Appendix

Source codes of all applications are stored in folder /src, whole solution is build by docker-compose.

Also integration with CI/CD pipeline is not in scope of this document, because preparation of AKS cluster (include RBAC, istio configuration) has to be done first. Microservice deployment is done from command line by helm tool, helm can be integrated easily with any CI/CD pipeline.

## HPA (Horizontal POD Autoscaller)

### Create deployment in testing namespace

```bash
# create namespace
kubectl create namespace hpatest

# create ReplicaSet with service
cat <<EOF | kubectl apply --namespace hpatest -f -
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: webstress
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: webstress
    spec:
      containers:
        - name: hostname
          image: valda/webstress
          env:
          - name: CYCLECOUNT
            value: "500"
          resources:
            limits:
              cpu: 100m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: webstress
  name: webstress
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
  selector:
    app: webstress
  type: LoadBalancer
EOF

# wait for external IP of service
kubectl get svc --namespace hpatest

# create HPA
kubectl autoscale rs webstress --namespace hpatest --min=1 --max=8 --cpu-percent=75

# monitor HPA
kubectl get hpa --namespace hpatest -w
```

### Test it

Now we can generate load to our end point, let's use `ab` command line tool (you can install it by `sudo apt install apache2-utils').

```bash
ab -k -c 8 -n 200000 http://[IP ADDRESS]/perf
```

Or you can run few times curl command and after test you can kill all running background curl commands by 'pkill'

```bash
# you can run fet times this background task ...
curl "[IP ADDRESS]/perf?x=[0-10000]" 2> /dev/null > /dev/null &

# after test you can kill all curl processes
pkill curl
```

### Cleanup

```bash
kubectl delete namespace hpatest
```

