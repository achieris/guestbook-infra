#  guestbook infra

This project contains the applicationsets, applications and placements needed to deploy the guestbook app on different environments using ACM and OpenShift GitOps.

The  *dev* and *prod* folders contains the application manifest that will be deployed on the target clusters. This resource will be processed by the remote ArgoCD instance that will deploy all the components of the application. 

The *rhacm* folder contains the applicationset used to generate on the cluster hub the applications responsible for the deploy of the remote applications.


```
.
├── dev
│   └── application.yaml
├── prod
│   └── application.yaml
└── rhacm
    ├── all-openshift-clusters.yaml
    ├── dev-appset.yaml
    ├── dev-placement.yaml
    ├── prod-appset.yaml
    └── prod-placement.yaml
```
## rhacm/<cluster_env>-appset.yaml and rhacm/<cluster_env>/placement.yaml manifests

The applicationset create automatically applications using the generators. 
In the following case we use the Cluster Decision Resource Generator. 
(https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster-Decision-Resource/)

```
kind: ApplicationSet
metadata:
  name: guestbook-<cluster_env>
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: <cluster_env>-placement
        requeueAfterSeconds: 180
  template:
    metadata:
      labels:
        velero.io/exclude-from-backup: "true"
      name: 'guestbook-{{name}}'
    spec:
      destination:
        namespace: openshift-gitops
        server: '{{server}}'
      project: default
      source:
        path: <cluster_env>
        repoURL: https://github.com/achieris/guestbook-infra.git
        targetRevision: main
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
        - PruneLast=true

```
This is the placement used in the applicationset generator.

```
kind: Placement
metadata:
  name: <cluster_env>-placement
  namespace: openshift-gitops
spec:
  predicates:
  - requiredClusterSelector:
      labelSelector:
        matchExpressions:
        - key: cluster_env
          operator: In
          values:
          - <cluster_env> 

```
## <cluster_env>/application.yaml manifests
This is the application resource deployed on the <cluster_env> remote clusters.
The syncPolicy is not automated as requested by the customer.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/achieris/guestbook.git
    targetRevision: main
    path: <cluster_env>
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook 
  syncPolicy:
    syncOptions:     
    - CreateNamespace=true 
```
