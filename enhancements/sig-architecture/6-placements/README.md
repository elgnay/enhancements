# New Placement APIs

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would define new placement APIs to select a set of managed clusters from multiple managed clustersets.

## Motivation

Since placementrule is used in multiple components (GRC/App/Observability). It is more appropriate to  define it in API repo and have a generic component to consume it based on the current placementrule apis in https://github.com/open-cluster-management/multicloud-operators-placementrule 

The placement API will provide a definition on how to select a set of managed clusters in multiple managed clustersets. And placement decisions will be generated according to the spec defined in the placement API.

Placement API can be used by a higher primitive and controller to decide which clusters the manifests should be placed.  


### Goals

- Define a new placement APIs in API repo to select a set of managed clusters in multiple managed clustersets;
- Have a generic component to consume placements and generate placement decisions;
- Limit the access to managed clusters by leveraging managed clustersets;

### Non-Goals

- Support tolerations/taints. ManagedCluster has no taint defined in spec yet;
- Create metrics data when creating/updating placement decisions. It needs further investigation and should be covered in a separate enhancement; 

## Proposal

### User Stories

#### Story 1: A cluster-admin on hub can place a policy or workload to all clusters in all clustersets; The cluster-admin on hub can also place a policy or workload to any selected clusters in clustersets.

#### Story 2: A normal user on hub can place a policy or a workload to a slice of clusters in one or multiple clustersets that the user is authorized to access.

#### Story 3: A user can use predicate to select clusters
  - Select openshift clusters only
  - Select clusters in aws and gcp
  - Select cluster not in vsphere
  - Select openshift clusters with a specific version
  - Select clusters labeled as prod (which is set by users).

#### Story 4: A user can define the number of clusters to be selected
  - Select 3 openshift cluster
  - Select 5 cluster on aws

#### Story 5: The placement API should be scalable to support selecting hundreds of clusters from thousands of clusters.

#### Story 6: A user can select clusters across different regions.

#### Story 7: A controller is able to consume placements to select desired clusters.

#### Story 8: A user can specify tolerations and select clusters with particular taints.

#### Story 9: A user can fetch metrics data and figure out how the decision is made for a placement.

#### Story 10: A user can select clusters with certian conditions
   - Select available clusters

### API Spec

#### Placement API
`Placement` is a namespace scoped API and represents a selection of clusters in one or multiple managedclustersets. When a placement is created, a controller consuming the placement should generate one or multiple placement decisions based on the spec defined in placement.
```go
// Placement defines a rule to select a set of managed clusters from the 
// clustersets bound to the placement namespace.
// 
// User is able to bind a clusterset to a namespace in two ways:
// 1) Create a ManagedClusterSetBinding in the namespace if they have an RBAC 
//    rule to CREATE on the virtual subresource of `managedclustersets/bind`;
// 2) Create a service account with name 'placement', and create a 
//    ClusterRoleBinding to grant the CREATE permission on the virtual subresource
//    `managedclustersets/bind` of a clusterset to this service account;
//
// A slice of PlacementDecisions with label placement.open-cluster-management.io={placement name}
// will be created to represent the cluster selected by this placement.
 
type Placement struct {
   metav1.TypeMeta   `json:",inline"`
   metav1.ObjectMeta `json:"metadata,omitempty"`
 
   // Spec defines the attributes of Placement.
   Spec PlacementSpec `json:"spec"`
   
   // Status represents the current status of the Placement
   // +optional
   Status PlacementStatus `json:"status,omitempty"`
}
 
// PlacementSpec defines the attributes of Placement.
// The result of ClusterSets, NumberOfClusters, Predicates and AntiAffinity
// are ANDed. An empty PlacementSpec selects all clusters from the bound clustersets
// to placement namespace.
type PlacementSpec struct {
   // ClusterSets represent the ManagedClusterSets from which the clusters are 
   // selected. If the slice is empty, clusters will be selected from the 
   // clustersets bound to the placement namespace, otherwise clusters will be 
   // selected from the intersection of this slice and the clustersets bound to the 
   // placement namespace.
   // +optional
   ClusterSets []string `json:"clusterSets,omitempty"`
 
   // NumberOfClusters represents the number of clusters to be selected in this 
   // placement.
   // nil indicates that NumberOfClusters is not set and there is no bound number
   // +optional
   NumberOfClusters *int32 `json:"numberOfClusters,omitempty"`
 
   // Predicates represent a slice of predicates to select clusters. The predicates 
   // are ORed.
   // +optional
   Predicates []ClusterPredicate `json:"predicates,omitempty"`
 
   // AntiAffinity represents constraints among the selected clusters
   // +optional
   AntiAffinity AntiAffinity `json:"antiAffinity,omitempty"`
}
 
// ClusterPredicate represents a predicate to select clusters. The result of 
// LabelSelector, ClusterName and ClaimSelector are ANDed.
type ClusterPredicate struct {
   // LabelSelector represents a selector of clusters by label
   // +optional
   LabelSelector metav1.LabelSelector `json:"labelSelector"`
 
   // Name represents a selector of clusters by name
   // +optional
   ClusterName string `json:"clusterName"`
 
   // ClaimSelector represents a selector of cluster by cluster claims
   // +optional
   ClaimSelector ClusterClaimSelector `json:"claimSelector"`
}
 
// A cluster claim selector is a claim query over a set of managed clusters. 
// The result of matchClaims and matchExpressions are ANDed. An empty cluster claim
// selector matches all objects. A null cluster claim selector matches no objects.
type ClusterClaimSelector struct {
   // matchClaims is a map of {key,value} pairs. A single {key,value} in 
   // the matchClaims map is equivalent to an element of matchExpressions, 
   // whose key field is "key", the operator is "In", and the values array
   // contains only "value". The requirements are ANDed.
   // +optional
   MatchClaims map[string]string `json:"matchClaims,omitempty"`
   // matchExpressions is a list of cluster claim selector requirements. 
   // The requirements are ANDed.
   // +optional
   MatchExpressions []metav1.LabelSelectorRequirement `json:"matchExpressions,omitempty"`
}
 
// A placement with an AntiAffinity specified must ensure all selected clusters
// have different values of topology key, which is either a cluster label or a
// cluster claim with name TopologyKey.
type AntiAffinity struct {
   // TopologyKey is either a label key or a cluster claim name of clusters
   // +required
   TopologyKey string `json:"topologyKey"`
 
   // TopologyKeyType indicates the type of TopologyKey. It could be Label or Claim.
   // +required
   TopologyKeyType string `json:"topologyKeyType"`
}
 
const (
   TopologyKeyTypeLabel string = "Label"
   TopologyKeyTypeClaim string = "Claim"
)
 
type PlacementStatus struct {
   // Conditions contain the different condition statuses for this Placement.
   Conditions []metav1.Condition `json:"conditions"`
}
 
const (
   // PlacementConditionSatisfied means Placement is satisfied.
   // A placement is not satisfied only if
   // 1) NumberOfClusters is specified;
   // 2) And the number of matched clusters is less than NumberOfClusters;
   PlacementConditionSatisfied string = "PlacementSatisfied"
)
```

#### PlacementDecision API
`PlacementDecision` represents a slice of clusters selected by a placement. A `Placement` can link to multiple `PlacementDecisions`, and uses a label selector for reference.
```go
// PlacementDecision indicates a decision from a placement
// PlacementDecision should has a label placement.open-cluster-management.io={placement name}
// to reference a certain placement.
type PlacementDecision struct {
   metav1.TypeMeta   `json:",inline"`
   metav1.ObjectMeta `json:"metadata,omitempty"`
 
   // Decisions is a slice of decisions according to a placement
   // The number of decisions should not be larger than 100
   Decisions []ClusterDecision `json:"decisions"`
}
 
// ClusterDecision represents a decision from a placement
type ClusterDecision struct {
   // ClusterName is the name of the managedcluster
   // +required
   ClusterName string `json:"clusterName"`
 
   // Available represents whether a cluster is available.
   // +required
   Available bool `json:"available"`
}
```

## Design Details

### User Scenarios
#### Scenario 1: A normal user creates clustersetbindings manually to limit the scope of the clustersets a placement can access.
1. A normal user creates a namespace;
    ```
    kubectl create ns ns1
    ```

2. The user lists all authorized clustersets;
    ```
    kubectl get managedclustersets.view.cluster.open-cluster-management.io
    ```

3. The user creates clustersetbindings to bind some of the authorized clustersets to the namespace. An example looks like,
    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: ManagedClusterSetBinding
    metadata:
      name: clusterset1
      namespace: ns1
    spec:
      clusterSet: clusterset1
    ```
4. The user creates a placement in the namespace;
    ```yaml
    apiVersion: placement.open-cluster-management.io/v1alpha1
    kind: Placement
    metadata:
      name: placement1
      namespace: ns1
    spec:
      predicates:
        - labelSelector:
            matchLabels:
              vendor: OpenShift
    ```

5. PlacementController generates placementdecisions in the same namespace according to the clustersetbindings in this namespace and the spec of the placement.
    ```yaml
    apiVersion: placement.open-cluster-management.io/v1alpha1
    kind: PlacementDecision
    metadata:
      labels:
        placement.open-cluster-management.io: placement1
      name: placement1-decision-1
      namespace: ns1
    spec:
      decisions:
        - clusterName: cluster1
          available: true
        - clusterName: cluster2
          available: false
        - clusterName: cluster3
          available: true
    ```

Limitations:
- A clustersetbinding binds only one clusterset to a namespace. It becomes worse if a user, like admin, wants to bind all clustersets to a namespace, because,
  - The number of clustersets might be very large;
  - The new added clustersets cannot be bound to the namespace automatically;

#### Scenario 2: Cluster admin create clusterrolebindings for a particular service account to limit the scope of the clustersets a placement can access.

1. Cluster admin creates a namespace;
    ```
    kubectl create ns ns1
    ```

2. Cluster admin creates a service account ‘placement’ in the namespace;
    ```
    kubectl -n ns1 create sa placement
    ```

3. Cluster admin creates one or multiple clusterrolebindings to grant this service account the permissions to access some clustersets; A clusterrolebinding looks like,
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: open-cluster-management:managedclusterset:clusterset1:ns1:placement
    subjects:
    - kind: ServiceAccount
      name: placement
      namespace: ns1
    roleRef:
      kind: ClusterRole
      name: open-cluster-management:managedclusterset:clusterset1
      apiGroup: rbac.authorization.k8s.io
    ```

4. ClusterSetBindingController monitors clusterrolebindings of the service account ‘placement’, and creates clustersetbinding for each clusterset in the referenced clusterrole;

5. Cluster admin creates a placement in the namespace;
    ```yaml
    apiVersion: placement.open-cluster-management.io/v1alpha1
    kind: Placement
    metadata:
      name: placement1
      namespace: ns1
    spec:
      predicates:
        - labelSelector:
            matchLabels:
              vendor: OpenShift
          claimSelector:
            matchExpressions:
              - key: version.openshift.io
                operator: In
                values:
                  - 4.5.8
    ```

6. PlacementController generates placementdecisions in the same namespace according to the clustersetbindings in this namespace and the spec of the placement.
    ```yaml
    apiVersion: placement.open-cluster-management.io/v1alpha1
    kind: PlacementDecision
    metadata:
      labels:
        placement.open-cluster-management.io: placement1
      name: placement1-decision-1
      namespace: ns1
    spec:
      decisions:
        - clusterName: cluster1
          available: true
        - clusterName: cluster2
          available: false
        - clusterName: cluster3
          available: true
    ```

In step 3 above, clusterrolebinding is created instead of rolebindings because,
- A rolebinding does not take effect at all if it refers to a clusterrole which defines the permissions on cluster-scope resources, like clustersets.
- There exists a security hole if a user creates a rolebinding with reference to a clusterrole of an unauthorized clusterset.
