# Secure DevOps and Secure AKS cluster

## Abstract

Aim of this repo is to prepare environment and process for secure development and secure operation of solution in DevOps environment for agile teams. Solution is build on top of AKS (kubernetes cluster) with istio service mesh extension and supporting container solution for specific Azure services (like KeyVault).

Lets try to deliver rules and boundaries for different roles in DevOps and organization operation teams.

Solution uses for demonstration set of microservices in different languages (AngularJS frontend, .NET Core, Java spring-boot, NodeJS) and utilizes different Azure services (AKS, Azure Container Registry, Azure SQL Database, Azure Database for PostgreSQL, CosmosDB - with MongoDB API, KeyVault, Application Insights)

### Solution architecture

Picture describes architecture design of our microservice solution and also defines communication flows for components.

### Roles definition

* AKS administrator (the one who is able configure all AKS assets)
* Security Admin (setup istio rules for all namespaces)
* Solution level (namespace level)
    * Security (setting up credentials for DB and security rules for istio)
    * Network (define ingress/egress rules and routing)
    * Deployment (deploy solution)
    * Operation (see logs and component health - read permission to objects)

## Prerequisites 

Installed tools:
* kubectl - https://kubernetes.io/docs/tasks/tools/install-kubectl/
* helm - https://docs.helm.sh/using_helm/#installing-helm
* istio 1.0.2 - https://istio.io/docs/setup/kubernetes/download-release/
* az CLI - https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest


