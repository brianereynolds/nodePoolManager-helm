# nodepoolmanager
Node Pool Manager is used to create, update and delete node pools on an AKS cluster.

## Description
Some applications cannot take full advantage of the Kubernetes high-availability concepts. This CRD has been designed to manage the node pools that run application workloads.

Based on the YAML you provide, the controller will bring the node pools to the state reflected in the configuration.

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


### Contributions
This controller has been written using kubebuilder https://book.kubebuilder.io

I would love to have some contributors to the project. Contact me on github

### Uninstall
Uninstall the helm chart
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