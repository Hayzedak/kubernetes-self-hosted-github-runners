# GitHub Actions Runner Controller (ARC) on kubernets Setup Guide

This guide walks through setting up the GitHub Actions Runner Controller (ARC) on kubernetes.

## Prerequisites

- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- GitHub account

## 1. GitHub Configuration

### 1.1 Create a GitHub Organization

1. Log into GitHub
2. Click the '+' icon in the top-right corner
3. Select "New organization"
4. Choose the free plan
5. Fill in the organization details and create

### 1.2 Generate a Fine-Grained Personal Access Token

1. Go to GitHub Settings > Developer settings > Personal access tokens
2. Click on "Fine-grained tokens"
3. Click "Generate new token"
4. Configure the token:
   - Name: "ARC Access Token" (or your preferred name)
   - Expiration: Set as needed
   - Description: Optional, but recommended
   - Resource owner: Select your newly created organization
   - Repository access: "All repositories" (or select specific ones)
   - Permissions:
     - Repository permissions:
       - Actions: Read and Write
       - Administration: Read and Write
       - Contents: Read and Write
       - Metadata: Read-only (auto-selected)
     - Organization permissions:
       - Self-hosted runners: Read and Write
5. Generate and immediately copy the token

## 2. Minikube Setup

### 1. Start Minikube:

```
minikube start
```

### 2. Verify Minikube status:

```
minikube status
```

## 3. Install cert-manager

cert-manager is a prerequisite for ARC.

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
kubectl wait --for=condition=Ready pods -n cert-manager --all --timeout=60s
```

## 4. Install Actions Runner Controller

### 4.1 Install ARC using kubectl

```
kubectl create -f https://github.com/actions/actions-runner-controller/releases/download/v0.25.2/actions-runner-controller.yaml
```

### 4.2 Create Secret for GitHub Token

```
kubectl create secret generic controller-manager \
    -n actions-runner-system \
    --from-literal=github_token=YOUR_FINE_GRAINED_TOKEN
```

Replace `YOUR_FINE_GRAINED_TOKEN` with your actual token.

### 4.3 Verify Installation

```
kubectl get pods -n actions-runner-system
```

## 5. Create a Runner Deployment

Create a file named `runner-deployment.yaml`:

```
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: hng-runnerdeploy
spec:
  replicas: 1
  template:
    spec:
      organization: "hayzed-org"
      labels:
        - self-hosted
        - linux
        - x64
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: hng-runner-autoscaler
spec:
  scaleTargetRef:
    name: example-runnerdeploy
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - hayzed-org/boilerplate_golang_web
```       

Replace YOUR_GITHUB_ORG with your GitHub organization name, also make sure to replace the org name and repo name to yours.


Apply the deployment:

```
kubectl apply -f runner-deployment.yaml
```

## 6. Verify Runner Registration

Check runner pods:

```
kubectl get pods -n actions-runner-system
```

## 7. Using Your Self-Hosted Runner
In your GitHub Actions workflow YAML in your specified org repo, specify the runner:

```
jobs:
  build:
    runs-on: self-hosted
```

8. Troubleshooting

Check ARC logs:

```
kubectl logs -n actions-runner-system deployment/controller-manager
```

Describe runner pods:

```
kubectl describe pods -n actions-runner-system
```

Check runners:

```
kubectl get runners
```

9. Cleanup

To uninstall ARC:

```
kubectl delete -f https://github.com/actions/actions-runner-controller/releases/download/v0.25.2/actions-runner-controller.yaml
kubectl delete namespace actions-runner-system
```
