# gitops-apps

This repository stores Argo CD application definitions and environment-specific deployment state.

The goal is to keep GitOps concerns separate from:
- application source code in `fastapi-app`
- reusable Helm charts in `helm-charts`
- cluster/bootstrap infrastructure in `tf-minikube`

## Repository purpose

This repo answers the question:
- what should be deployed
- to which environment
- with which values

In the first version, it manages one environment:
- `minikube`

And one application stack:
- `fastapi-app`

## Repository layout

- `bootstrap/minikube/root-application.yaml`: root Argo CD application for the minikube environment
- `apps/minikube/`: child Argo CD applications managed by the root application
- `manifests/minikube/fastapi-app-prereqs/`: raw Kubernetes manifests needed before the Helm chart syncs
- `environments/minikube/fastapi-app/values.yaml`: values file for the `fastapi-app` Helm chart in the `minikube` environment

## How this setup works

This repo uses an app-of-apps pattern.

### Root application

`bootstrap/minikube/root-application.yaml` points Argo CD at `apps/minikube` in this repo.
That allows Argo CD to manage the environment by reading application definitions from Git.

### Prerequisites application

`apps/minikube/00-fastapi-prereqs.yaml` installs:
- the `fastapi-app` namespace
- the `fastapi-app-db` Secret

This runs first so the Helm chart has the namespace and credentials it expects.
The ordering is controlled with Argo CD sync waves: prerequisites use wave `0`, and the app uses wave `1`.

### fastapi-app application

`apps/minikube/10-fastapi-app.yaml` is a multi-source Argo CD application:
- source 1: the `fastapi-app` Helm chart from `helm-charts`
- source 2: the environment values from this `gitops-apps` repo

That separation is intentional:
- chart logic lives in `helm-charts`
- deployment intent and environment values live here

## Current source revisions

Right now the `fastapi-app` application points to:
- `helm-charts` repo revision: `main`
- `gitops-apps` repo revision: `main`

Why:
- `helm-charts` is expected to contain the chart on `main`
- `gitops-apps` is expected to be merged to `main` before Argo CD bootstraps from it

In practice, the steady-state expectation is that Argo CD reads both repos from `main`.

## Credentials note

For the local `minikube` environment, this repo stores a plain Kubernetes `Secret` manifest:
- `manifests/minikube/fastapi-app-prereqs/secret.yaml`

That is acceptable for a local/dev bootstrap, but not something I would keep for higher environments.

Later, for real environments, you would typically replace this with one of:
- External Secrets
- Sealed Secrets
- SOPS
- another secret-management integration

## fastapi-app chart values for minikube

The minikube environment currently deploys:
- image tag `v1.0.1`
- bundled PostgreSQL enabled
- DB credentials read from the `fastapi-app-db` Secret

Those values live in:
- `environments/minikube/fastapi-app/values.yaml`

## Bootstrap flow

Prerequisite: Argo CD must already be installed in the cluster.
That is what `tf-minikube` is for.

If Terraform in `tf-minikube` is configured with `gitops_bootstrap_enabled=true`, this bootstrap step is created automatically.

If you ever need to bootstrap manually, use:

```bash
kubectl apply -f bootstrap/minikube/root-application.yaml -n argocd
```

Then verify:

```bash
kubectl get applications -n argocd
kubectl get pods -n fastapi-app
```

## Accessing fastapi-app in minikube

Once Argo CD has synced the applications, you can test the app with:

```bash
kubectl port-forward svc/fastapi-app -n fastapi-app 8000:8000
curl -X PUT http://127.0.0.1:8000/hello/Alice \
  -H 'Content-Type: application/json' \
  -d '{"dateOfBirth":"1992-02-02"}'
curl http://127.0.0.1:8000/hello/Alice
```

## How to update the deployed app version

A typical flow is:
1. publish a new `fastapi-app` image
2. update the Helm chart version or defaults in `helm-charts`
3. update `environments/minikube/fastapi-app/values.yaml` if you want a different image tag or environment-specific override
4. commit and push this repo
5. let Argo CD sync the change

## How to extend this repo later

As more apps or environments are added, keep this pattern:
- one folder under `apps/<environment>/` per Argo CD application definition
- one folder under `environments/<environment>/` per app values set
- raw manifests only for prerequisites or things that are not better expressed as a Helm chart

Examples you might add later:
- `apps/dev/`
- `apps/prod/`
- `environments/dev/fastapi-app/values.yaml`
- `environments/prod/fastapi-app/values.yaml`
