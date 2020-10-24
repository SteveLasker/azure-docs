---
title: How to manage public content in private registry
description: ....
ms.topic: article
ms.date: 10/21/2020
---

# How to manage public content with Azure Container Registry

An Azure container registry hosts your container images and other [OCI artifacts][oci-artifacts] in a private, authenticated environment. However, your environment may have dependencies on public content such as public container images or [helm charts][helm-charts]. For example, you might run [nginx] for service routing or `docker build FROM `[alpine][alpine-public-image] by pulling images directly from Docker Hub or another public registry, but don't have robust processes to maintain them. For more information about the risks introduced by dependencies on public content and best practices see the [OCI Consuming Public Content Blog][oci-consuming-public-content].

This article covers features and workflows in Azure Container Registry to help you manage consuming and maintain public content:

* Import local copies of dependent public images.
* Validate public images through security scanning and functional testing.
* Promoting to private registries for internal usage.
* Triggering base image updates for applications dependent upon public content.
* Using [ACR Tasks](container-registry-tasks-overview.md) to automate this workflow.

#TODO:[!Consuming Public Content Workflow](./media/container-registry-consuming-public-content/consuming-public-content-workflow.png)

This article refers mainly to container images, but concepts apply to other supported [registry content](container-registry-image-formats.md) such as Helm charts and other artifacts.

The gated import workflow refers to decoupling your organizations dependencies on externally managed artifacts. For instance, images sourced from public registries like: [docker hub][docker-hub], [gcr][gcr], [quay][quay], [github package registry][ghcr], [Microsoft Container Registry][mcr] or even other public [Azure Container Registries][acr].

Consider balancing these two, possibly conflicting goals:

* Do you really want an unexpected upstream change to possibly take out your production system?
* Do you want upstream security fixes, for the versions you depend upon, to be automatically deployed?

## Prerequisites

* Create a clone of docker hub for public images.
  * This allows us simulate a base image update, which would normally be initiated on docker hub.
* Create a central registry, which will host the base artifacts.
* Create a development team registry, that will host one more more teams that build and manage images
  * **Note:** [repository based RBAC (preview)][acr-repo-permissions] is now available, enabling multiple teams to share a single registry, with unique permission sets

### Set your environment variables

Set the three registry names:

```azurecli
REGISTRY_HUB=publichub
REGISTRY_HUB_RG=${REGISTRY_HUB}-rg
REGISTRY_HUB_URL=${REGISTRY_HUB}.azurecr.io
REGISTRY_BASE_ARTIFACTS=acmebaseartifacts
REGISTRY_BASE_ARTIFACTS_RG=${REGISTRY_BASE_ARTIFACTS}-rg
REGISTRY_BASE_ARTIFACTS_URL=${REGISTRY_BASE_ARTIFACTS}.azurecr.io
REGISTRY=acmerockets
REGISTRY_URL=${REGISTRY}.azurecr.io
REGISTRY_RG=${REGISTRY}-rg
RESOURCE_GROUP_LOCATION=eastus
```

GIT repositories and token

```azurecli
# NEW
GIT_BASE_IMAGE_NODE=https://github.com/importing-public-content/base-image-node.git#main
GIT_NODE_IMPORT=https://github.com/importing-public-content/import-baseimage-node.git#main
GIT_HELLO_WORLD=https://github.com/importing-public-content/hello-world.git#main

GIT_TOKEN_NAME=import-public-content
GIT_TOKEN=<set-git-token-here>
```

Azure Keyvault for storing secrets

```azurecli
AKV=importing
AKV_RG=${AKV}-rg
```

ACI for hosting the deployed application

```azurecli
ACI=hello-world-aci
ACI_RG=${ACI}-rg
```
App Service for hosting the deployed application

```azurecli
APP_SERVICE_URL=https://acme-rockets.azurewebsites.net
```

Configure az cli defaults

```azurecli
az configure --defaults acr=$REGISTRY
```

Create Registries

```azurecli
az group create --name $REGISTRY_HUB_RG --location $RESOURCE_GROUP_LOCATION
az acr create --resource-group $REGISTRY_HUB_RG --name $REGISTRY_HUB --sku Premium

az group create --name $REGISTRY_BASE_ARTIFACTS_RG --location $RESOURCE_GROUP_LOCATION
az acr create --resource-group $REGISTRY_BASE_ARTIFACTS_RG --name $REGISTRY_BASE_ARTIFACTS --sku Premium

az group create --name $REGISTRY_RG --location $RESOURCE_GROUP_LOCATION
az acr create --resource-group $REGISTRY_RG --name $REGISTRY --sku Premium
```

Create KeyVault for secrets

```azurecli
az group create --name $AKV_RG --location $RESOURCE_GROUP_LOCATION
az keyvault create --resource-group $AKV_RG --name $AKV
az keyvault secret set --vault-name $AKV --name $GIT_TOKEN_NAME --value $GIT_TOKEN
```

Verify KeyVault values

```azurecli
az keyvault secret show --vault-name $AKV --name $GIT_TOKEN_NAME --query value -o tsv
```

## Create public node base image

To simulate the node image on docker hub, create an ACR Task to build and maintain the public image

```azurecli
az acr task create \
  --name hub-node \
  -r $REGISTRY_HUB \
  -f Dockerfile \
  -t node:{{.Run.ID}} \
  -t node:9-alpine \
  --context $GIT_BASE_IMAGE_NODE \
  --git-access-token $(az keyvault secret show \
                        --vault-name $AKV \
                        --name $GIT_TOKEN_NAME \
                        --query value -o tsv)
```

Run the task to generate a build

```azurecli
az acr task run -r $REGISTRY_HUB -n hub-node
```

## Create a Token for access to the "public" registry

```azurecli
az keyvault secret set \
  --vault-name $AKV \
  --name "public-registry-user" \
  --value "public-registry-access"

az keyvault secret set \
  --vault-name $AKV \
  --name "public-registry-password" \
  --value $(az acr token create \
              --name public-registry-access \
              --registry $REGISTRY_HUB \
              --scope-map _repositories_pull \
              -o tsv \
              --query credentials.passwords[0].value)

az keyvault secret show \
  --vault-name $AKV \
  --name public-registry-user \
  --query value -o tsv

az keyvault secret show \
  --vault-name $AKV \
  --name public-registry-password \
  --query value -o tsv

## Import the public image into the base artifacts registry

```azurecli
az acr import \
  --name $REGISTRY_BASE_ARTIFACTS \
  --source ${REGISTRY_HUB_URL}/node:9-alpine \
  --image node:9-alpine \
  --username $(az keyvault secret show \
                --vault-name $AKV \
                --name public-registry-user \
                --query value -o tsv) \
  --password $(az keyvault secret show \
                --vault-name $AKV \
                --name public-registry-password \
                --query value -o tsv) \
  --force
  
  az acr task create \
  --name base-import \
  -f acr-task.yaml \
  -r $REGISTRY_BASE_ARTIFACTS \
  --context $GIT_BASE_IMAGE_NODE \
  --git-access-token $(az keyvault secret show \
                        --vault-name $AKV \
                        --name $GIT_TOKEN_NAME \
                        --query value -o tsv)
```

## Create the hello-world image

```azurecli
PRINCIPAL_ID=$(az acr task create \
  -n hello-world \
  -r $REGISTRY \
  -t hello-world:{{.Run.ID}} \
  -f Dockerfile \
  --arg=REGISTRY_URL=${REGISTRY_BASE_ARTIFACTS_URL}/ \
  --context $GIT_HELLO_WORLD \
  --git-access-token $(az keyvault secret show \
                        --vault-name $AKV \
                        --name $GIT_TOKEN_NAME \
                        --query value -o tsv) \
  --assign-identity \
  -o tsv --query identity.principalId)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --scope $(az acr show --name $REGISTRY_BASE_ARTIFACTS --query id --output tsv) \
  --role AcrPull

az acr task credential add \
  --name hello-world \
  --registry $REGISTRY \
  --login-server $REGISTRY_BASE_ARTIFACTS_URL \
  --use-identity [system]
```

```azurecli
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --scope $(az acr show --name ${REGISTRY_BASE_ARTIFACTS} \
              --query id --output tsv) \
  --role AcrPull
```

Build the `hello-world` image to verify the task permissions are functioning

```azurecli
az acr task run -r $REGISTRY -n hello-world
```

## Create an ACR Token for access by ACI

```azurecli
ACR_TOKEN_PASSWORD=$(az acr token create \
  --name $GIT_TOKEN_NAME \
  --registry $REGISTRY \
  --repository hello-world \
  content/read content/write \
  -o json \
  --query "credentials.passwords[0].value")
```

Save the token to Keyvault

```azurecli
az keyvault secret set \
  --vault-name $AKV \
  --name "aci-pull-user" \
  --value "aci-pull"

az keyvault secret set \
  --vault-name $AKV \
  --name "aci-pull-password" \
  --value $(az acr token create \
              --name "aci-pull" \
              --registry $REGISTRY \
              --scope-map _repositories_pull \
              -o tsv \
              --query credentials.passwords[0].value)
```

## Create an ACI Instance

```azurecli
az group create --name $ACI_RG --location $RESOURCE_GROUP_LOCATION
HELLO_WORLD_TAG=dat
az container create \
  --resource-group $ACI_RG \
  --name ${ACI} \
  --image ${REGISTRY_URL}/hello-world:$HELLO_WORLD_TAG \
  --registry-username=$(az keyvault secret show \
                        --vault-name $AKV \
                        --name "aci-pull-user" \
                        --query value -o tsv) \
  --registry-password=$(az keyvault secret show \
                        --vault-name $AKV \
                        --name "aci-pull-password" \
                        --query value -o tsv) \
  --ip-address=public \
  --ports 80

az container show \
  --resource-group $ACI_RG \
  --name ${ACI} \
  --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
  --out table
```

Open the FQDN in the browser

## Manually update the base image

Open the `Dockerfile` in base-image-node repo
Change the `BACKGROUND_COLOR` to `Red`:

```Dockerfile
ARG REGISTRY_NAME=
FROM ${REGISTRY_NAME}node:9-alpine
ENV NODE_VERSION 9.1-alpine
ENV BACKGROUND_COLOR Red
```

Commit the change and watch for ACR Tasks to automatically start building:

```azurecli
watch -n1 az acr task list-runs -r $REGISTRY_HUB
```

You should eventually see STATUS `Succeeded` based on a TRIGGER of `Commit`:

```azurecli
RUN ID    TASK      PLATFORM    STATUS     TRIGGER    STARTED               DURATION
--------  --------  ----------  ---------  ---------  --------------------  ----------
ca4       hub-node  linux       Succeeded  Commit     2020-10-24T05:02:29Z  00:00:22
```

## Manually import the updated base image

```azurecli
az acr import \
  --name $REGISTRY_BASE_ARTIFACTS \
  --source ${REGISTRY_HUB_URL}/node:9-alpine \
  --image node:9-alpine \
  --username $(az keyvault secret show \
                --vault-name $AKV \
                --name public-registry-user \
                --query value -o tsv) \
  --password $(az keyvault secret show \
                --vault-name $AKV \
                --name public-registry-password \
                --query value -o tsv) \
  --force
```

Watch for ACR Tasks to automatically start the hello-world image:

```azurecli
watch -n1 az acr task list-runs -r $REGISTRY
```

You should eventually see STATUS `Succeeded` based on a TRIGGER of `Image Update`

```azurecli
RUN ID    TASK         PLATFORM    STATUS     TRIGGER       STARTED               DURATION
--------  -----------  ----------  ---------  ------------  --------------------  ----------
dau       hello-world  linux       Succeeded  Image Update  2020-10-24T05:08:45Z  00:00:31
```

## Manually update the ACI instance

```azurecli
HELLO_WORLD_TAG=dau
az container create \
  --resource-group $ACI_RG \
  --name ${ACI} \
  --image ${REGISTRY_URL}/hello-world:$HELLO_WORLD_TAG \
  --registry-username=$(az keyvault secret show \
                        --vault-name $AKV \
                        --name "aci-pull-user" \
                        --query value -o tsv) \
  --registry-password=$(az keyvault secret show \
                        --vault-name $AKV \
                        --name "aci-pull-password" \
                        --query value -o tsv) \
  --ip-address=public \
  --ports 80
```

## Stopping red backgrounds

We've now completed blah blah blah

Our next step is to automate the importing of node base image from the public registry to our base artifacts registry.
As part of the import, we'll stop any red backgrounds from coming through with validation tests

========
STOPPING POINT
========

## Image import

Use [image import](container-registry-import-images.md) in your workflows to import Docker images from public registries easily to your Azure container registries. The [az acr import](/cli/azure/acr)
Basic, manual feature to move/import base images to ACR from multiple public clouds  (other ACR, DH, GCR, quay, etc.)

Import to (or georeplicate to) same region where you'll deploy - "bring the content to the region"

## Maintain base images in separate registry(ies)

"Regardless of the size of the company, you'll likely want to have a separate registry for managing base images. While it's possible to share a registry with multiple development teams, it's difficult to know how each team may work, possibly requiring VNet features, or other registry specific capabilities.

Depending on your workflow, you may want two registries for base images, one for staging and validating base images, and one for storing verified base images ready for use by your dev teams.

Decouple management of base images from consumption by dev teams.

## Adopt tagging scheme for base image updates

See container-registry-image-tag-version.md

* Build images from stable service tags - can continue to receive security patches and framework updates.

## Protect images using Image/tag locking

Support your build and deployment workflows from unintentional image updates.

https://docs.microsoft.com/en-us/azure/container-registry/container-registry-image-lock

* Lock repo from new content
* Block pulls or deletes

## Automate base image updates

Expanding on basic image import, they should create a process by which they import the base images they depend upon into their corporate registry.

1. Automate base image updates by buffering to a staging registry, tests are run as part of the build process. If the tests succeed, the mirrored image is imported to a base-images repository. Use ACR tasks to automate base artifact validation:

* Pull to staging registry - build both "mirrored" image and "test" image, including test/validation scripts
* Scan [build image to run a vulnerability scanner such as Aqua]
* Test [build image to run a test script]
* If validated, move to base images reg repo

NEED --> Example multistep task to accomplish this? something like
https://github.com/Azure-Samples/acr-tasks/blob/master/build-test-update-hello-world.yaml or https://github.com/SteveLasker/Presentations/tree/master/demo-scripts/buffering-building-patching

## Trigger downstream app image builds based on base image update

Using ACR Tasks, tracking base image updates in the base reg repo:

Concepts: container-registry-tasks-base-images.md

Walkthroughs: https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-base-image-update (base in same reg)
https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-private-base-image-update (base in separate reg)

## Next steps

See Sample E2E scenario

https://github.com/SteveLasker/Presentations/tree/master/demo-scripts/buffering-building-patching

========

 and your might host copies of certain public images for your organization's deployments.

Dependencies on public content can introduce numerous risks to your organization
 
[Intro] Main message is: Use ACR to keep local copies of all artifacts that organization depends on for development and deployments.
 
Example: Companies should never base their FROM statements on external registries
 
Should mitigate Docker TOS changes but also a best practice

[acr]:                          https://aka.ms/acr
[oci-artifacts]:                https://aka.ms/acr/artifacts
[helm-charts]:                  https://helm.sh
[oci-consuming-public-content]: https://docs.google.com/document/d/1fxayMznIkszBI9Y2S3KGSyi2hFMwUIwDfn3D2wQcye4/edit?usp=sharing
[nginx-public-image]:           https://hub.docker.com/_/nginx
[alpine-public-image]:          https://hub.docker.com/_/alpine
[ghcr]:                         https://docs.github.com/en/free-pro-team@latest/packages/getting-started-with-github-container-registry/about-github-container-registry
[quay]:                         https://quay.io
[docker-hub]:                   https://hub.docker.com
[gcr]:                          https://cloud.google.com/container-registry
[mcr]:                          http://aka.ms/mcr
[acr-repo-permissions]:         https://aka.ms/acr/repo-permissions