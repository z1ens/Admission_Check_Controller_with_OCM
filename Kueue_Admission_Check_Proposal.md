
# Proposal for External Kueue Admission Check Controller

## Introduction

### Existing Components

1. **OCM Placement and AddonPlacementScore**:
   - Open-Cluster-Management (OCM) uses placement and AddonPlacementScore to select clusters.
   - It generates a PlacementDecision that lists the selected clusters.

2. **MultiKueue for Job Dispatching**:
   - Kubernetes has a MultiKueue feature for job dispatching in multiple clusters.
   - The MultiKueue Admission Check Controller on the manager cluster manages Kueue and allows Kueue to consider additional criteria before admitting a Workload.
   - Kueue only accepts a workload if all AdmissionChecks provide a positive signal.

### Objective
Integrate OCM placement results with MultiKueue by creating an Admission Check Controller that incorporates the placement results into the Admission Check parameters.

## Proposal

### API Design

The Admission Check Controller will read the OCM placement results and generate corresponding MultiKueueConfig and MultiKueueCluster. Here is an example of the API design for the Admission Check Controller:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: AdmissionCheck
metadata:
  name: ocm-multikueue
spec:
  controllerName: open-cluster-management.io/placement
  parameters:
    apiGroup: cluster.open-cluster-management.io
    kind: Placement
    name: nvidia-t4
```

### Workflow

1. **Placement Decision Integration**:
   - The controller reads the placement decision from OCM.
   - It uses this information to populate the parameters of the Admission Check.

2. **Admission Check Controller**:
   - The controllerName specifies the OCM placement as the source of the decision.
   - The parameters include details such as the API group, kind, and name of the placement.

3. **MultiKueueConfig and MultiKueueCluster Creation**:
   - For each managed cluster selected in the placement decision, the controller generates a MultiKueueConfig and MultiKueueCluster.
   - These configurations include necessary details such as secrets for accessing the managed clusters.

### Benefits

- **Universal API Interface**: Provides a universal API interface for different clusters.
- **Automated MultiKueue Environment Setup**: Automatically generates MultiKueueConfig and MultiKueueCluster for each managed cluster, facilitating the setup of the MultiKueue environment.
- **Efficient Job Dispatching**: Utilizes OCM placement-selected clusters for job dispatching, ensuring that jobs are efficiently dispatched to clusters capable of handling them.

## Prototype Outline

### Step 1: Define the Admission Check Controller

Create a new Kubernetes custom resource definition (CRD) for the Admission Check Controller.

\`\`\`yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: admissionchecks.kueue.x-k8s.io
spec:
  group: kueue.x-k8s.io
  versions:
    - name: v1beta1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: admissionchecks
    singular: admissioncheck
    kind: AdmissionCheck
    shortNames:
      - ac
\`\`\`

### Step 2: Implement the Admission Check Controller Logic

Implement the controller logic to read the OCM placement decision and populate the parameters of the Admission Check.

\`\`\`go
package controllers

import (
    "context"
    "sigs.k8s.io/controller-runtime/pkg/client"
    kueuev1beta1 "kueue.x-k8s.io/api/v1beta1"
    ocmv1 "open-cluster-management.io/api/placement/v1"
)

type AdmissionCheckReconciler struct {
    client.Client
}

func (r *AdmissionCheckReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var placement ocmv1.Placement
    if err := r.Get(ctx, req.NamespacedName, &placement); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    var admissionCheck kueuev1beta1.AdmissionCheck
    admissionCheck.Spec.ControllerName = "open-cluster-management.io/placement"
    admissionCheck.Spec.Parameters = map[string]string{
        "apiGroup": "cluster.open-cluster-management.io",
        "kind":     "Placement",
        "name":     placement.Name,
    }

    if err := r.Create(ctx, &admissionCheck); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
\`\`\`

### Step 3: Deploy the Controller

Deploy the controller to the Kubernetes cluster.

\`\`\`yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-check-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admission-check-controller
  template:
    metadata:
      labels:
        app: admission-check-controller
    spec:
      containers:
        - name: manager
          image: your-repo/admission-check-controller:latest
          command:
            - /manager
          args:
            - --leader-elect
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 9443
              name: webhook-server
              protocol: TCP
\`\`\`

### Step 4: Validate the Implementation

1. **Create Placement Resources**: Create OCM placement resources to simulate cluster selection.
2. **Check Admission Controller**: Verify that the Admission Check Controller reads the placement decisions and creates the appropriate MultiKueueConfig and MultiKueueCluster resources.
3. **Job Dispatching**: Ensure that jobs are dispatched to the selected clusters, and remote jobs are deleted from other clusters when a job starts execution.
