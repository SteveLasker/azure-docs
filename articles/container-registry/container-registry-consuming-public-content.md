---
title: How to manage public content in private registry
description: ....
ms.topic: article
ms.date: 10/21/2020
---

# How to consume & maintain public content with Azure Container Registry Tasks

An Azure container registry hosts your container images and other [OCI artifacts][oci-artifacts] in a private, authenticated environment. However, your environment may have dependencies on public content such as public container images, [helm charts][helm-charts], [OPA][opa] Policies or other artifacts. For example, you might run [nginx] for service routing or `docker build FROM `[alpine][alpine-public-image] by pulling images directly from Docker Hub or another public registry. As upstream changes occur, this article will explain how to import and maintain these public artifacts.

For more information about the risks introduced by dependencies on public content and best practices see the [OCI Consuming Public Content Blog post][oci-consuming-public-content].

This article covers features and workflows in Azure Container Registry to help you manage consuming and maintaining public content:

======

TODO: UPDATE TOC

======

1. Simulate a Public Registry
2. Automate building a hello-world image
3. Automate deploying to an [Azure Container Instance][aci]
4. Simulate upstream changes directly to your environment
5. Create a gated import, that validates upstream changes are appropriate for your environment


* Import local copies of dependent public images.
* Validate public images through security scanning and functional testing.
* Promoting to private registries for internal usage.
* Triggering base image updates for applications dependent upon public content.
* Using [ACR Tasks](container-registry-tasks-overview.md) to automate this workflow.

![Consuming Public Content Workflow](./media/container-registry-consuming-public-content/consuming-public-content-workflow.png)

This article refers mainly to container images, but the concepts apply to other supported [registry artifacts](container-registry-image-formats.md).

The gated import workflow refers to decoupling your organizations dependencies on externally managed artifacts. For instance, images sourced from public registries like: [docker hub][docker-hub], [gcr][gcr], [quay][quay], [github container registry][ghcr], [Microsoft Container Registry][mcr] or even other public [Azure Container Registries][acr].

Consider balancing these two, possibly conflicting goals:

1. Do you really want an unexpected upstream change to possibly take out your production system?
2. Do you want upstream security fixes, for the versions you depend upon, to be automatically deployed?

## Prerequisites

* Create a clone of docker hub for public images.
  * This allows us simulate a base image update, which would normally be initiated on docker hub.
* Create a central registry, which will host the base artifacts.
* Create a development team registry, that will host one more more teams that build and manage images
  * **Note:** [repository based RBAC (preview)][acr-repo-permissions] is now available, enabling multiple teams to share a single registry, with unique permission sets

### Set your environment variables

In this section, we'll configure variables unique to your environment.

Set the three registry names:

```azurecli
REGISTRY_PUBLIC=publicregistry
REGISTRY_PUBLIC_RG=${REGISTRY_PUBLIC}-rg
REGISTRY_PUBLIC_URL=${REGISTRY_PUBLIC}.azurecr.io

REGISTRY_BASE_ARTIFACTS=acmebaseartifacts
REGISTRY_BASE_ARTIFACTS_RG=${REGISTRY_BASE_ARTIFACTS}-rg
REGISTRY_BASE_ARTIFACTS_URL=${REGISTRY_BASE_ARTIFACTS}.azurecr.io

REGISTRY=acmerockets
REGISTRY_RG=${REGISTRY}-rg
REGISTRY_URL=${REGISTRY}.azurecr.io

RESOURCE_GROUP_LOCATION=eastus
```

```
GIT repositories and token

To simulate your environment, fork each of these into repositories you can mange. Then, update the variables for your forked repositories.
Notice `:main` concatenated to the end of the git URLs representing the default repository branch.

```azurecli

GIT_BASE_IMAGE_NODE=https://github.com/importing-public-content/base-image-node.git#main
GIT_NODE_IMPORT=https://github.com/importing-public-content/import-baseimage-node.git#main
GIT_HELLO_WORLD=https://github.com/importing-public-content/hello-world.git#main
```

Establish a Git Token for ACR Tasks to clone and establish git webhooks. 
See: @DAN, CAN YOU UPDATE TO A REFERENCE FOR REQUIRED PERMISSIONS?

```azurecli
GIT_TOKEN_NAME=import-public-content
GIT_TOKEN=<set-git-token-here>
```

Azure KeyVault for storing secrets

```azurecli
AKV=acr-task-credentials
AKV_RG=${AKV}-rg
```

ACI for hosting the deployed application

```azurecli
ACI=hello-world-aci
ACI_RG=${ACI}-rg
```

Create Registries

```azurecli
az group create --name $REGISTRY_PUBLIC_RG --location $RESOURCE_GROUP_LOCATION
az acr create --resource-group $REGISTRY_PUBLIC_RG --name $REGISTRY_PUBLIC --sku Premium

az group create --name $REGISTRY_BASE_ARTIFACTS_RG --location $RESOURCE_GROUP_LOCATION
az acr create --resource-group $REGISTRY_BASE_ARTIFACTS_RG --name $REGISTRY_BASE_ARTIFACTS --sku Premium

az group create --name $REGISTRY_RG --location $RESOURCE_GROUP_LOCATION
az acr create --resource-group $REGISTRY_RG --name $REGISTRY --sku Premium
```

Create KeyVault for secrets

```azurecli
az group create --name $AKV_RG --location $RESOURCE_GROUP_LOCATION
az keyvault create --resource-group $AKV_RG --name $AKV
```

Verify KeyVault values

```azurecli
az keyvault secret set --vault-name $AKV --name $GIT_TOKEN_NAME --value $GIT_TOKEN
az keyvault secret show --vault-name $AKV --name $GIT_TOKEN_NAME --query value -o tsv
```

Create a Resource Group for an Azure Container Instance

```azurecli
az group create --name $ACI_RG --location $RESOURCE_GROUP_LOCATION
```

## Create public node base image

To simulate the node image on docker hub, create an [ACR Task][acr] to build and maintain the public image. This allows simulating changes by the node image maintainers

#TODO: Add GitHub Public Token

```azurecli
az acr task create \
  --name node-public \
  -r $REGISTRY_PUBLIC \
  -f acr-task.yaml \
  -t node:{{.Run.ID}} \
  -t node:15-alpine \
  --context $GIT_BASE_IMAGE_NODE \
  --git-access-token $(az keyvault secret show \
                        --vault-name $AKV \
                        --name $GIT_TOKEN_NAME \
                        --query value -o tsv)
```

Run the task to generate a build

```azurecli
az acr task run -r $REGISTRY_PUBLIC -n node-public
```

List the image in the simulated public registry

```azurecli
az acr repository show-tags -n $REGISTRY_PUBLIC --repository node
```

## Create a Token for access to the "public" registry

```azurecli
az keyvault secret set \
  --vault-name $AKV \
  --name "registry-${REGISTRY_PUBLIC}-user" \
  --value "registry-${REGISTRY_PUBLIC}-user"

az keyvault secret set \
  --vault-name $AKV \
  --name "registry-${REGISTRY_PUBLIC}-password" \
  --value $(az acr token create \
              --name "registry-${REGISTRY_PUBLIC}-user" \
              --registry $REGISTRY_PUBLIC \
              --scope-map _repositories_pull \
              -o tsv \
              --query credentials.passwords[0].value)
```

## Create an ACR Token for access by ACI to pull the image

A token to the registry with `hello-world` is created. Permissions are scoped to read (pull)

```azurecli
az keyvault secret set \
  --vault-name $AKV \
  --name "registry-${REGISTRY}-user" \
  --value "registry-${REGISTRY}-user"

az keyvault secret set \
  --vault-name $AKV \
  --name "registry-${REGISTRY}-password" \
  --value $(az acr token create \
              --name "registry-${REGISTRY}-user" \
              --registry $REGISTRY \
              --repository hello-world content/read \
              -o tsv \
              --query credentials.passwords[0].value)
```

## Create the `hello-world` image from the public registry

As we're simulating a public registry, which could be docker hub, we provide credentials in the form of docker login within the `acr-task.yaml`. Since the registry is an ACR, we use the token created above. However, this same pattern may be used to pass docker credentials to docker hub.

Wihin the `acr-task.yaml`, we deploy the newly build image to ACI. The resource group was created above. By calling `az container create` with only a difference in the `image:tag`, the same instance is used.

```azurecli
az acr task create \
  -n hello-world \
  -r $REGISTRY \
  -t hello-world:{{.Run.ID}} \
  -f acr-task.yaml \
  --context $GIT_HELLO_WORLD \
  --git-access-token $(az keyvault secret show \
                        --vault-name $AKV \
                        --name $GIT_TOKEN_NAME \
                        --query value -o tsv) \
  --set REGISTRY_FROM_URL=${REGISTRY_PUBLIC_URL}/ \
  --set-secret REGISTRY_FROM_USER=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY_PUBLIC}-user" \
      --query value -o tsv) \
  --set-secret REGISTRY_FROM_PASSWORD=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY_PUBLIC}-password" \
      --query value -o tsv) \
  --set-secret REGISTRY_USER=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY}-user" \
      --query value -o tsv) \
  --set-secret REGISTRY_PASSWORD=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY}-password" \
      --query value -o tsv) \
  --set ACI=$ACI \
  --set ACI_RG=$ACI_RG \
  --assign-identity
```

The task was assigned a system identity, which is given access to the ACI resource group for permissions to create the instance.

```azurecli
az role assignment create \
  --assignee $(az acr task show \
  --name hello-world \
  --registry $REGISTRY \
  --query identity.principalId --output tsv) \
  --scope $(az group show -n $ACI_RG --query id -o tsv) \
  --role owner
```

With the task created, run the task to build/deploy the hello-world image:

```azurecli
az acr task run -r $REGISTRY -n hello-world
```

Browse the site

```bash
explorer.exe "http://"$(az container show \
  --resource-group $ACI_RG \
  --name ${ACI} \
  --query ipAddress.ip \
  --out tsv)
```

## Update the base image with a "bad" change

Open the `Dockerfile` in base-image-node repo
Change the `BACKGROUND_COLOR` to `Red`:

```Dockerfile
ARG REGISTRY_NAME=
FROM ${REGISTRY_NAME}node:15-alpine
ENV NODE_VERSION 9.1-alpine
ENV BACKGROUND_COLOR Red
```

Commit the change and watch for ACR Tasks to automatically start building:

```azurecli
watch -n1 az acr task list-runs -r $REGISTRY_PUBLIC
az acr task logs -r $REGISTRY_PUBLIC
```

You should eventually see STATUS `Succeeded` based on a TRIGGER of `Commit`:

```azurecli
RUN ID    TASK      PLATFORM    STATUS     TRIGGER    STARTED               DURATION
--------  --------  ----------  ---------  ---------  --------------------  ----------
ca4       hub-node  linux       Succeeded  Commit     2020-10-24T05:02:29Z  00:00:22
```

Type `CTRL-C` to exit the watch command.

Watch for ACR Tasks to automatically start the hello-world image:

```azurecli
watch -n1 az acr task list-runs -r $REGISTRY
az acr task logs -r $REGISTRY
```

You should eventually see STATUS `Succeeded` based on a TRIGGER of `Image Update`

```azurecli
RUN ID    TASK         PLATFORM    STATUS     TRIGGER       STARTED               DURATION
--------  -----------  ----------  ---------  ------------  --------------------  ----------
dau       hello-world  linux       Succeeded  Image Update  2020-10-24T05:08:45Z  00:00:31
```

Type `CTRL-C` to exit the watch command.

## Checking in

At this point, you've created a hello-world image that is automatically built on git commits, or changes to the base image. While we've built against a base image in ACR, this could be any supported registry. If using Docker Hub, you would have to wait for someone to change the public image.

## Stopping red backgrounds

Our next step is to automate the importing of node base image from the public registry to our base artifacts registry.
As part of the import, we'll stop any red backgrounds from coming through with validation tests

```azurecli
  az acr task create \
  --name base-import-node \
  -f acr-task.yaml \
  -r $REGISTRY_BASE_ARTIFACTS \
  --context $GIT_NODE_IMPORT \
  --git-access-token $(az keyvault secret show \
                        --vault-name $AKV \
                        --name $GIT_TOKEN_NAME \
                        --query value -o tsv) \
  --set REGISTRY_FROM_URL=${REGISTRY_PUBLIC_URL}/ \
  --set-secret REGISTRY_FROM_USER=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY_PUBLIC}-user" \
      --query value -o tsv) \
  --set-secret REGISTRY_FROM_PASSWORD=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY_PUBLIC}-password" \
      --query value -o tsv) \
  --set REGISTRY=$REGISTRY \
  --assign-identity
```

The task was assigned a system identity, which is given access to the base-artifacts registry for permissions to import the tested base image.

```azurecli
az role assignment create \
  --assignee $(az acr task show \
                --name base-import-node \
                --registry $REGISTRY_BASE_ARTIFACTS \
                --query identity.principalId --output tsv) \
  --scope $(az acr show -n $REGISTRY_BASE_ARTIFACTS --query id -o tsv) \
  --role owner
```

## Troubleshooting

- `./test.sh: Permission denied`
      ```bash
      chmod +x ./test.sh
      ```

Add a `AcrPull` token to access the base-artifacts registry

```azurecli
az keyvault secret set \
  --vault-name $AKV \
  --name "registry-${REGISTRY_BASE_ARTIFACTS}-user" \
  --value "registry-${REGISTRY_BASE_ARTIFACTS}-user"

az keyvault secret set \
  --vault-name $AKV \
  --name "registry-${REGISTRY_BASE_ARTIFACTS}-password" \
  --value $(az acr token create \
              --name "registry-${REGISTRY_BASE_ARTIFACTS}-user" \
              --registry $REGISTRY_BASE_ARTIFACTS \
              --repository node content/read \
              -o tsv \
              --query credentials.passwords[0].value)
```

Update the `hello-world` image to pull from the base artifacts registry

```azurecli
az acr task update \
  -n hello-world \
  -r $REGISTRY \
  --set REGISTRY_FROM_URL=${REGISTRY_BASE_ARTIFACTS_URL}/ \
  --set-secret REGISTRY_FROM_USER=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY_BASE_ARTIFACTS}-user" \
      --query value -o tsv) \
  --set-secret REGISTRY_FROM_PASSWORD=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY_BASE_ARTIFACTS}-password" \
      --query value -o tsv) \
  --set-secret REGISTRY_USER=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY}-user" \
      --query value -o tsv) \
  --set-secret REGISTRY_PASSWORD=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY}-password" \
      --query value -o tsv) \
  --set ACI=$ACI \
  --set ACI_RG=$ACI_RG


az acr task run -r $REGISTRY -n hello-world
```

#TODO: Log bug on updating existing SET variables clears out previous values

Monitor base image updates

``azurecli
$REGISTRY=acmerockets

watch -n1 az acr task list-runs -r $REGISTRY_PUBLIC
az acr task logs -r $REGISTRY_PUBLIC

watch -n1 az acr task list-runs -r $REGISTRY_BASE_ARTIFACTS
az acr task logs -r $REGISTRY_BASE_ARTIFACTS

watch -n1 az acr task list-runs -r $REGISTRY
az acr task logs -r $REGISTRY
```

## Cleaning up

```azurecli
az group delete -n $REGISTRY_RG -y
az group delete -n $REGISTRY_PUBLIC_RG -y
az group delete -n $REGISTRY_BASE_ARTIFACTS_RG -y
az group delete -n $AKV_RG -y
az group delete -n $ACI_RG -y
```

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


## Extra Snippets

**NOTE:** _Many variables are out of sync_

az acr task run -n base-import-node -r $REGISTRY_BASE_ARTIFACTS
az acr task list-runs -r $REGISTRY_BASE_ARTIFACTS

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

Create the ACI

```azurecli
az group create --name $ACI_RG --location $RESOURCE_GROUP_LOCATION

az container create \
  --resource-group $ACI_RG \
  --name ${ACI} \
  --image ${REGISTRY_URL}/hello-world:$HELLO_WORLD_TAG \
  --registry-username $(az keyvault secret show \
                        --vault-name $AKV \
                        --name ${REGISTRY}-user \
                        --query value -o tsv) \
  --registry-password $(az keyvault secret show \
                        --vault-name $AKV \
                        --name ${REGISTRY}-password \
                        --query value -o tsv) \
  --ip-address=public \
  --ports 80 \
  -o jsonc
```

```azurecli
az acr task create \
  -n hello-world \
  -r $REGISTRY \
  -t hello-world:{{.Run.ID}} \
  -f acr-task-build.yaml \
  --set REGISTRY_URL=${REGISTRY_PUBLIC_URL}/ \
  --set-secret REGISTRY_PUBLIC_USER=$(az keyvault secret show \
      --vault-name $AKV \
      --name "registry-${REGISTRY_PUBLIC}-user" \
      --query value -o tsv) \
  --set-secret REGISTRY_PUBLIC_PASSWORD=$(az keyvault secret show \
      --vault-name $AKV \
      --name registry-${REGISTRY_PUBLIC}-password \
      --query value -o tsv) \
  --context $GIT_HELLO_WORLD \
  --git-access-token $(az keyvault secret show \
                        --vault-name $AKV \
                        --name $GIT_TOKEN_NAME \
                        --query value -o tsv) \
  --assign-identity
```

## Create an ACI Instance

Retrieve the latest tag for the hello-world image
```azurecli
HELLO_WORLD_TAG=$(az acr task list-runs \
  -r $REGISTRY \
  -n hello-world \
  -o tsv --query [0].name)
```

## Import the public image into the base artifacts registry

```azurecli
az acr import \
  --name $REGISTRY_BASE_ARTIFACTS \
  --source ${REGISTRY_PUBLIC_URL}/15-alpine \
  --image node:15-alpine \
  --username $(az keyvault secret show \
                --vault-name $AKV \
                --name "registry-${REGISTRY_PUBLIC}-user" \
                --query value -o tsv) \
  --password $(az keyvault secret show \
                --vault-name $AKV \
                --name "registry-${REGISTRY_PUBLIC}-password" \
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

## Manually import the updated base image

```azurecli
az acr import \
  --name $REGISTRY_BASE_ARTIFACTS \
  --source ${REGISTRY_PUBLIC_URL}/node:15-alpine \
  --image node:15-alpine \
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

[acr]:                          https://aka.ms/acr
[acr-repo-permissions]:         https://aka.ms/acr/repo-permissions
[acr-task]:                     https://aka.ms/acr/tasks
[aci]:                          http://aka.ms/aci
[alpine-public-image]:          https://hub.docker.com/_/alpine
[docker-hub]:                   https://hub.docker.com
[gcr]:                          https://cloud.google.com/container-registry
[ghcr]:                         https://docs.github.com/en/free-pro-team@latest/packages/getting-started-with-github-container-registry/about-github-container-registry
[helm-charts]:                  https://helm.sh
[mcr]:                          http://aka.ms/mcr
[nginx-public-image]:           https://hub.docker.com/_/nginx
[oci-artifacts]:                https://aka.ms/acr/artifacts
[oci-consuming-public-content]: https://docs.google.com/document/d/1fxayMznIkszBI9Y2S3KGSyi2hFMwUIwDfn3D2wQcye4/edit?usp=sharing
[quay]:                         https://quay.io
