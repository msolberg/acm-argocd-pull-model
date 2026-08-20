# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository contains Kubernetes/OpenShift YAML manifests for integrating ArgoCD (via OpenShift GitOps Operator) with Red Hat Advanced Cluster Management (ACM). It implements the ACM-to-ArgoCD cluster registration pattern.

## Architecture

The integration connects three ACM/OCM resources to automatically register ACM-managed clusters as ArgoCD cluster secrets:

1. **ManagedClusterSetBinding** — Grants the ArgoCD namespace (`openshift-gitops`) permission to access clusters in a ManagedClusterSet.
2. **Placement** — Selects which managed clusters from the bound ClusterSet should be registered with ArgoCD.
3. **GitOpsCluster** — Bridges ACM and ArgoCD by referencing a Placement and an ArgoCD instance; ACM creates/maintains ArgoCD cluster secrets for each placed cluster.

All three resources must be co-located in the same namespace as the ArgoCD instance because GitOpsCluster references the Placement by name (namespace-scoped).

## Commands

Apply the manifests in dependency order:
```
oc apply -f managedclustersetbinding.yaml -f placement.yaml -f gitopscluster.yaml
```

Verify the integration:
```
oc get managedclustersetbinding -n openshift-gitops
oc get placement -n openshift-gitops
oc get gitopscluster -n openshift-gitops
oc get secret -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster
```

The last command confirms that ACM has auto-created ArgoCD cluster secrets for the placed clusters.
