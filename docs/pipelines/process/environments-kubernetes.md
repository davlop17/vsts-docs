---
title: Environment - Kubernetes resource
description: Kubernetes resource views within Environment
ms.topic: reference
ms.assetid: b318851c-4240-4dc2-8688-e70aba1cec55
ms.manager: atulmal
ms.date: 05/03/2019
monikerRange: azure-devops
---

# Environment - Kubernetes resource
[!INCLUDE [include](../includes/version-team-services.md)]

Kubernetes resource view within environments provides a glimpse of the status of objects within the namespace mapped to the resource. It also overlays pipeline traceability on top of these objects so that one can trace back from a Kubernetes object to the pipeline and then back to the commit.

## Overview

The advantages of using Kubernetes resource views within environments include - 
- **Pipeline traceability** - The [Kubernetes manifest task](../tasks/deploy/kubernetes-manifest.md) used for deployments adds additional annotations to portray pipeline traceability in resource views. This can help in identifying the originating Azure DevOps organization, project and pipeline responsible for updates made to an object within the namespace.

  > [!div class="mx-imgBorder"]
  > ![Pipeline traceability](media/k8s-pipeline-traceability.png)

- **Diagnose resource health** - Workload status can be useful in quickly debugging potential mistakes or regressions that could have been introduced by a new deployment. For example, in the case of unconfigured *imagePullSecrets* resulting in ImagePullBackOff errors, pod status information can help identify the root cause for this issue.
  > [!div class="mx-imgBorder"]
  > ![ImagePullBackOff](media/k8s-imagepullbackoff.png)

- **Review App** - Review app works by deploying every pull request from Git repository to a dynamic Kubernetes resource under the environment. Reviewers can see how those changes look as well as work with other dependent services before they’re merged into the target branch and deployed to production.

## Kubernetes resource creation

### Azure Kubernetes Service

A [ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) is created in the chosen cluster and namespace. For an RBAC enabled cluster, [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions) is created as well to limit the scope of the created service account to the chosen namespace. For an RBAC disabled cluster, the ServiceAccount created has cluster-wide privileges (across namespaces).

1. In the environment details page, click on **Add resource** and choose **Kubernetes**.
2. Select **Azure Kubernetes Service** in the Provider dropdown.
3. Choose the Azure subscription, cluster and namespace (new/existing).
4. Click on **Validate and create** to create the Kubernetes resource.

### Using existing service account

While the Azure Provider option creates a new ServiceAccount, the generic provider allows for using an existing ServiceAccount to allow a Kubernetes resource within environment to be mapped to a namespace.

> [!TIP]
> Generic provider (existing service account) is useful for mapping a Kubernetes resource to a namespace from a non-AKS cluster.

1. In the environment details page, click on **Add resource** and choose **Kubernetes**.
2. Select **Generic provider (existing service account)** in the Provider dropdown.
3. Input cluster name and namespace values.
4. For fetching Server URL, execute the following command on your shell:

   ```
   kubectl config view --minify -o 'jsonpath={.clusters[0].cluster.server}'
   ```
5. For fetching Secret object required to connect and authenticate with the cluster, the following sequence of commands need to be run:

   ```
   kubectl get serviceAccounts <service-account-name> -n <namespace> -o 'jsonpath={.secrets[*].name}'
   ```   

   The above command fetches the name of the secret associated with a ServiceAccount. The output of the above command is to be substituted in the following command for fetching Secret object: 

   ```
   kubectl get secret <service-account-secret-name> -n <namespace> -o json
   ```

   Copy and paste the Secret object fetched in JSON form into the Secret text-field.

6. Click on **Validate and create** to create the Kubernetes resource.

## Setup Review App

Below is an example YAML snippet for adding Review App to an **existing** pipeline. In this example, the first deployment job is run for non-PR branches and performs deployments against regular Kubernetes resource under environments. The second job runs only for PR branches and deploys against review app resources (namespaces inside Kubernetes cluster) generated on the fly. These resources are marked with a 'Review' label in the resource listing view of the environment.

```YAML
- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build

  jobs:
  - deployment: Deploy
    condition: and(succeeded(), not(startsWith(variables['Build.SourceBranch'], 'refs/pull/')))
    displayName: Deploy
    pool:
      vmImage: $(vmImageName)
    environment: $(envName).$(resourceName)
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: Create imagePullSecret
            inputs:
              action: createSecret
              secretName: $(imagePullSecret)
              dockerRegistryEndpoint: $(dockerRegistryServiceConnection)
              
          - task: KubernetesManifest@0
            displayName: Deploy to Kubernetes cluster
            inputs:
              action: deploy
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yml
                $(Pipeline.Workspace)/manifests/service.yml
              imagePullSecrets: |
                $(imagePullSecret)
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)

  - deployment: DeployPullRequest
    displayName: Deploy Pull request
    condition: and(succeeded(), startsWith(variables['Build.SourceBranch'], 'refs/pull/'))
    pool:
      vmImage: $(vmImageName)
      
    environment: '$(envName).$(k8sNamespaceForPR)'
    strategy:
      runOnce:
        deploy:
          steps:
          - reviewApp: $(resourceName)

          - task: Kubernetes@1
            displayName: 'Create a new namespace for the pull request'
            inputs:
              command: apply
              useConfigurationFile: true
              inline: '{ "kind": "Namespace", "apiVersion": "v1", "metadata": { "name": "$(k8sNamespaceForPR)" }}'

          - task: KubernetesManifest@0
            displayName: Create imagePullSecret
            inputs:
              action: createSecret
              secretName: $(imagePullSecret)
              namespace: $(k8sNamespaceForPR)
              dockerRegistryEndpoint: $(dockerRegistryServiceConnection)
          
          - task: KubernetesManifest@0
            displayName: Deploy to the new namespace in the Kubernetes cluster
            inputs:
              action: deploy
              namespace: $(k8sNamespaceForPR)
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yml
                $(Pipeline.Workspace)/manifests/service.yml
              imagePullSecrets: |
                $(imagePullSecret)
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)
```

For setting up review apps without the need to author the above YAML from scratch, checkout new pipeline creation experience using the [Deploy to Azure Kubernetes Services template](../ecosystems/kubernetes/aks-template.md)
