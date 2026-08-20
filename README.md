# ArgoCD Pull Model with Advanced Cluster Management

This repository contains the manifests needed to integrate ArgoCD (OpenShift GitOps) with Red Hat Advanced Cluster Management (ACM) using the pull model.

## Push vs Pull Model

In the **push model**, the hub ArgoCD connects directly to managed clusters and pushes resources to them. This requires network connectivity from the hub to each managed cluster's API server.

In the **pull model**, each managed cluster runs its own ArgoCD instance. The hub ArgoCD generates Application CRs via ApplicationSets, ACM wraps them as ManifestWork and distributes them to managed clusters, and each cluster's local ArgoCD reconciles the Application against its own API server. No direct network path from hub to managed cluster is required.

## Prerequisites

- OpenShift cluster with the following operators installed:
  - Red Hat Advanced Cluster Management for Kubernetes
  - Red Hat OpenShift GitOps (ArgoCD)
- One or more managed clusters registered with ACM

## Manifests

### Core Integration (required)

#### `managedclustersetbinding.yaml`

Binds the `default` ManagedClusterSet to the `openshift-gitops` namespace. This grants Placements in the ArgoCD namespace permission to select clusters from the set. Without this binding, the Placement cannot reference clusters in the `default` ClusterSet.

#### `placement.yaml`

Selects which managed clusters should participate in the GitOps integration. This Placement selects all OpenShift clusters from the `default` ClusterSet. It must live in the `openshift-gitops` namespace because the GitOpsCluster references it by name (namespace-scoped).

#### `gitopscluster.yaml`

Bridges ACM and ArgoCD by referencing the Placement and the ArgoCD instance. ACM uses this to create and maintain ArgoCD cluster secrets for each cluster matched by the Placement.

### Pull Model Add-On (required for pull model)

#### `managedclusteraddon-cluster1.yaml`

Deploys the `gitops-addon` to the `cluster1` managed cluster. This installs the pull model agent, which watches for ManifestWork resources containing ArgoCD Application CRs and applies them to the managed cluster's local ArgoCD instance.

#### `managedclusteraddon-local-cluster.yaml`

Deploys the `gitops-addon` to the hub cluster (`local-cluster`). This may not be necessary if the hub already has ArgoCD running and you only need pull model behavior on remote managed clusters.

### Optional Configuration

#### `addondeploymentconfig.yaml`

Customizes how the `gitops-addon` is deployed on managed clusters. The example sets the ArgoCD namespace to `openshift-gitops`. Only needed if you want to override the addon's default configuration.

## Usage

Apply the core integration resources first, then the pull model add-ons:

```bash
# Core integration
oc apply -f managedclustersetbinding.yaml
oc apply -f placement.yaml
oc apply -f gitopscluster.yaml

# Pull model add-on (for each managed cluster)
oc apply -f managedclusteraddon-cluster1.yaml
```

## Namespace Permissions

OpenShift GitOps is namespace-scoped by default. The ArgoCD application controller on each managed cluster must be granted access to any namespace it manages. Label target namespaces on the managed cluster so the GitOps operator creates the necessary RoleBindings:

```bash
oc label namespace <target-namespace> argocd.argoproj.io/managed-by=openshift-gitops
```

If the namespace is created as part of your Application manifests, include the label in the Namespace resource:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
```

Without this label, syncs will fail with "is forbidden" errors for the application controller service account.

## Verification

```bash
# Confirm the ClusterSet is bound
oc get managedclustersetbinding -n openshift-gitops

# Confirm the Placement has scheduled decisions
oc get placement -n openshift-gitops

# Confirm the GitOpsCluster is created
oc get gitopscluster -n openshift-gitops

# Confirm ArgoCD cluster secrets were created
oc get secret -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster

# Confirm the gitops-addon is available on managed clusters
oc get managedclusteraddon -A | grep gitops-addon
```
