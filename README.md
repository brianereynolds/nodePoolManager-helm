# nodepoolmanager
Node Pool Manager is used to create, update and delete node pools on an AKS cluster.

## Description
Some applications cannot take full advantage of Kubernetes high-availability concepts. This CRD has been designed to manage the node pools that run application workloads.

Based on the YAML you provide, the controller will bring the node pools to the state reflected in the configuration.

The CRD can be useful for managing different versions of node pools. This allows the AKS control plan to be upgraded independently of the node pools and subsequently move workloads around to newly upgraded node pools. 

To list the valid node pool versions using Azure CLI, you can use the az aks get-upgrades command

```
az aks get-upgrades --resource-group <resource-group> --name <cluster-name>
```
[Reference](https://learn.microsoft.com/en-us/azure/aks/manage-node-pools)

To upgrade control plan only:
```
az aks upgrade \
   --resource-group <ResourceGroupName> --name <AKSClusterName> \
   --control-plane-only \
   --kubernetes-version <KubernetesVersion>
```
[Reference](https://learn.microsoft.com/en-us/azure/architecture/operator-guides/aks/aks-upgrade-practices)


## Getting Started

### Prerequisites
- Access to a Azure Kubernetes Service 1.26+
- A Managed Identity with AKS Contributor Role for the cluster

### Deployment
Helm chart :
```
kubectl create ns operations
helm repo add k8smanagers https://k8smanagers.blob.core.windows.net/helm/
helm install nodepoolmanager k8smanagers/nodepoolmanager -n operations
```

If custom values are required in the helm chart, create a values.yaml and including with -f on helm command. Supported

* private image repository
* custom image tag
* custom image pull secrets
* custom labels for Deployment
* additional environment variables (see security section below)

Example:
```yaml
controllerManager:
  deployment:
    customLabels:                                # Custom labels
      custom1: value1
      custom2: value2
  manager:
    repository: my-image-repo/node-pool-manager  # Private image repo
    tag: specific-tag                            # Custom image tag
imagePullSecrets:
  - name: my-image-pull-secret                   # Custom image pull secret
```

#### Verification
```
helm list --filter 'nodepoolmanager' -n operations
kubectl get crd nodepoolmanagers.k8smanagers.greyridge.com
```

### Configuration
To activate the CRD configuration, apply a YAML
```yaml
apiVersion: k8smanagers.greyridge.com/v1
kind: NodePoolManager
metadata:
  labels:
    app.kubernetes.io/name: nodepoolmanager
    app.kubernetes.io/managed-by: kustomize
  name: nodepoolmanager-sample
spec:
  # Subscription ID of the cluster
  subscriptionId: "3e54eb54-xxxx-yyyy-zzzz-d7b190cd45cf"

  # Resource group containing the cluster
  resourceGroup: "node-upgrader"

  # The name of the cluster
  clusterName: "lm-cluster"

  # When this flag is true, if an error occurs, the CRD will ask Kubernetes to retry it automatically
  # This will result in the CRD being re-executed with the current configuration.
  # When this flag is false, the CRD will only be executed when the configuration is new/changed.
  retryOnError: false

  # When this flag is true, no changes will be executed, only logged
  testMode: false
  
  # A list of node pools to manage
  # If the node pool does not exist - it is created
  # If the node pool exists - it is updated to reflect the configuration
  # If the node pool exists and the action is "delete" - the node pool is deleted
  nodePools:
    # The name of the nodepool. The name of a node pool may only contain lowercase alphanumeric characters and must begin with a lowercase letter.
    - name: "greenpool"
      
      # The action to apply to the node pool. Choose one, optional
      # Action: "create|update|delete"
      
      # A list of properties for this node pool. See "ManagedClusterAgentPoolProfileProperties" Microsoft or Go reference for more details
      properties: {
        # Node pool version to create/update:
        #     Must be V.v.n. 
        #     Must must have the same major version as the control plane.
        #     Minor version must be within two minor versions of the control plane version
        #     Cannot be greater than the control plane version
        #     For more information see [https://docs.microsoft.com/azure/aks/use-multiple-node-pools#upgrade-a-node-pool].
        orchestratorVersion: "1.28.5",

        # VM size availability varies by region
        vmSize: "Standard_DS2_v2",
        
        # Whether to enable auto-scaler
        enableAutoScaling: true,
        
        # The minimum number of nodes for auto-scaling
        minCount: 0,
        
        # The maximum number of pods that can run on a node
        maxCount: 1,
        
        # The list of Availability zones to use for nodes.
        availabilityZones: [ "1" ]
      }
    - name: "bluepool"
      properties: {
        orchestratorVersion: "1.28.5",
        vmSize: "Standard_DS2_v2",
        enableAutoScaling: true,
        minCount: 0,
        maxCount: 1,
        availabilityZones: [ "2" ]
      }
```

#### Other Considerations
* Any node pools which are not listed in the YAML will be ignored.
* System pools _cannot_ be created, however they _can_ be upgraded

### Security
The controller users DefaultAzureCredential. The DefaultAzureCredential is a feature of the Azure Identity library that simplifies 
authentication for applications running on Azure. It automatically selects the best available credential based on the environment 
it's running in, making it easier to manage authentication across different stages of development and deployment.

The controller has been tested with 
* ManagedIdentityCredential
* EnvironmentCredential (Enterprise app/Service Principal)

#### Service Principal
The SPN details can be set when installing the helm chart. Create a values.yaml and include with -f on helm command.

Example:

```kubectl create secret generic azure-client-secret --from-literal=AZURE_CLIENT_SECRET=myclientsecret -n operations ```

```yaml
controllerManager:
  deployment:
    env:
      - name: AZURE_CLIENT_ID
        value: aaa-bbb-ccc
      - name: AZURE_TENANT_ID
        value: xxx-yyy-zzz
      - name: AZURE_CLIENT_SECRET
        valueFrom:
          secretKeyRef:
            name: azure-client-secret
            key: AZURE_CLIENT_SECRET
```

Assign the role _Azure Kubernetes Service Contributor Role_ to the MI which matches the details above.

#### Managed Identity
During AKS provisioning, a "managed cluster" resource group will be created, named `MC_[rgroup]_[clustername]_[region]`. 
This resource group will contain a managed identity, usually named `[clustername]_agentpool`. 
Add an Azure role assignment to this MI. The role to add is _Azure Kubernetes Service Contributor Role_.

### Contributions
This controller has been written using kubebuilder https://book.kubebuilder.io

I would love to have some contributors to the project. Contact me on github

### Uninstall
To remove the instance
```
helm delete NodePoolManager nodepoolmanager-sample
```

To fully uninstall the helm chart
```
helm uninstall -n operations nodepoolmanager
```

## License

Copyright 2024.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
