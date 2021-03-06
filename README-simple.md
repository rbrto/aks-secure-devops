# DevOps and AKS cluster

*created by Valdemar Zavadsky (Microsoft)*

## Abstract

Aim of this repo is to prepare environment and process for development  of solution in DevOps environment for agile teams. Solution is build on top of AKS (kubernetes cluster).

Solution uses for demonstration set of microservices in different languages (AngularJS frontend in nginx, Java spring-boot, NodeJS) and utilizes different Azure services (AKS, Azure Container Registry, Azure Database for PostgreSQL, CosmosDB - with MongoDB API, Application Insights)

### Solution architecture
Application architecture:

* SPA - single page angular application which is hosted in nginx HTML pages
* ToDo - java-spring-boot application which stores ToDo items in PostgreSQL database, API are exposed on /api/todo endpoint.
* Log - internal microservice which collects data manipulation operations in services(in our case in ToDo service). It is called only from ToDo microservice.

## Prerequisites 

Installed tools:
* kubectl - https://kubernetes.io/docs/tasks/tools/install-kubectl/
* helm - https://docs.helm.sh/using_helm/#installing-helm
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

### Deploy AKS cluster

Deploy cluster with RBAC enabled.

```bash
export AKS_RESOURCE_GROUP=AKSSEC
export LOCATION="northeurope"
export AKS_CLUSTER_NAME=akssec

az aks create --resource-group ${AKS_RESOURCE_GROUP} --name ${AKS_CLUSTER_NAME} \
  --no-ssh-key --kubernetes-version 1.11.3 --node-count 3 --node-vm-size Standard_DS1_v2 \
  --location ${LOCATION}
```

```bash
az aks get-credentials --name ${AKS_CLUSTER_NAME} --resource-group ${AKS_RESOURCE_GROUP} 
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

## Initialize kubernetes cluster

Now we can download kubernetes credentials for admin and install helm.

```bash
# download kubernetes config for admin
az aks get-credentials --name ${AKS_CLUSTER_NAME} --resource-group ${AKS_RESOURCE_GROUP}

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
az aks get-credentials --name ${AKS_CLUSTER_NAME} --resource-group ${AKS_RESOURCE_GROUP}

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

