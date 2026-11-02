# AKS Store Demo Advanced GitOps

> [!IMPORTANT]  
> This document is not intended to represent final state or production best-practices for GitOps with Fleet Manager.

1. Create a Fleet Manager with a hub cluster.

1. Setup a group of clusters and add to the fleet. Label members with label `appgroup` set to `dev`, `test`, `prod` (you can also split prod into `proda` and `prodb` if you want). 

1. Make sure to enable the Flux extension on all clusters and the Fleet Manager hub cluster. You **don't** need to create a FluxConfig resource.

    Fleet:

    ```bash
    az k8s-extension create --name flux \
        --extension-type microsoft.flux \
        --cluster-name ${FLEET} \
        --resource-group ${GROUP} \
        --cluster-type fleets \
        --scope cluster
    ```

    AKS:

    ```bash
    az k8s-extension create --name flux \
        --extension-type microsoft.flux \
        --cluster-name ${CLUSTER_NAME} \
        --resource-group ${GROUP} \
        --cluster-type managedCluster \
        --scope cluster
    ```

1. Create `aks-store-demo` namespace on hub cluster. You can use the Azure portal to manually create a namespace, or "Apply a YAML" that defines the namespace, or use `kubectl` connected to the Fleet Manager hub cluster (get credentials by using `az fleet get-credentials` with an authorized user).

1. Distribute to only clusters with `appgroup` label. See [01-aks-store-namespace-crp.yaml](./manifests/01-aks-store-namespace-crp.yaml). Apply using the Azure portal by using "Apply a YAML" option from the Fleet Resources > Namespaces option under the fleet, or use `kubectl` to apply the YAML via the Kubernetes API. 

1. Create the necessary authorization for Flux to manipulate resources within the `aks-store-demo` namespace by applying the  [02-flux-authorization-hub-namespace.yaml](./manifests/02-flux-authorization-hub-namespace.yaml) file.

1. Apply the Flux authorization across all the member clusters as well by using resource placement to distribute. See [03-distribute-flux-authorization.yaml](./manifests/03-distribute-flux-authorization.yaml).

1. So we can control the rollout of the application, we will define a strategy by applying it to the hub cluster using the [04-app-staged-rollout-strategy.yaml](./manifests/04-app-staged-rollout-strategy.yaml) manifest applied on the hub cluster.

1. Next, let's wire up the GitOps deployment to bring the necessary Flux bits into the Fleet Manager hub cluster. Use [05-wire-up-gitops-hub.yaml](./manifests/05-wire-up-gitops-hub.yaml) to stage the `HelmRepository` and `HelmRelease` Flux resources on the hub cluster. Both are set to `suspend=true` so they don't attempt to reconcile on the Fleet Manager hub cluster.

1. The repository includes a Fleet Manager namespace-scoped resource placement ([06-aks-store-components-pick-clusters.yaml](./manifests/06-aks-store-components-pick-clusters.yaml)) that targets only clusters with the `appgroup` label, and defines a rollout strategy of `External` so we can use a StagedUpdateRun (staged rollout) to deploy the application across our clusters.

    1. Additionally, there is an Fleet Manager namespace-scope resource override ([07-override-components.yaml](./manifests/07-override-components.yaml)) to removes the `suspend=true` when the resoures are placed on matching member clusters.

1. Finally, to create the rollout apply the [08-rollout-app.yaml](./manifests/08-rollout-app.yaml) to the Fleet Manager hub cluster. Patch to the state to start the rollout, or use the Fleet Manager Azure portal experience to manually start.


