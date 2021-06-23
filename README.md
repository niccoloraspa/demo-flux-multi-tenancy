# demo-flux-multi-tenancy

Simple demo for managing multi-tenant clusters with Git and Flux v2.

## Tools

- fluxcd v2 `>= 0.15` ([installation guide](https://toolkit.fluxcd.io/guides/installation/))
- minikube `>= v1.18` ([installation guide](https://minikube.sigs.k8s.io/docs/start/))

## 0. GitHub Token

Create a GitHub personal access token following this [guide](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token).

Remember to grant all `repo` permissions.

Once created save the token in an environment variable:

```bash
export GITHUB_TOKEN=<TOKEN_ID>
export GITHUB_USER=<MY_GITHUB_USER> (e.g. nikever)
```

## 1. Start minikube cluster

1. At the root level of this repository, execute:

```bash
export REPO_DIR=$(PWD)
```

2. Start a minikube cluster:

```bash
cd $REPO_DIR/minikube
make setup
```

By default, it will spin up a one node Kubernetes cluster of version `v1.19.4` in a VirtualBox VM (CPUs=2, Memory=4096MB).

You can pass additional parameters to change the default values:

```bash
make setup cpu=4 memory=4096
```

Please referer to this [Makefile](minikube/Makefile) for additional details on the cluster creation.

## 1. Bootstrap Flux V2

1. Bootstrap Flux via the cli:

```bash
flux bootstrap github \
    --owner $GITHUB_USER \
    --repository flux-fleet \
    --branch main \
    --path ./clusters/minikube \
    --personal
```

The `flux bootstrap` command:

- creates a repository if one doesn't exist
- commits the manifests for the Flux components to the default branch at the specified path
- installs the Flux components
- configures the target cluster to synchronize with the specified path inside the repository

2. Wait for the pods to be up and running:

```bash
kubectl -n flux-system get pods -w
```

3. Clone the created repository:

```bash
mkdir $REPO_DIR/demo
cd $REPO_DIR/demo

git clone https://github.com/$GITHUB_USER/flux-fleet
cd flux-fleet
```

## 2. Deploy the sample Hello App via GitOps

1. Create a folder for the Hello App:

```bash
mkdir -p ./clusters/minikube/hello-app/
```

2. Create a `GitRepository` manifest pointing to the sample [*Hello App*](https://github.com/nikever/kubernetes-hello-app) repository's `main` branch:

```bash
flux create source git hello-app \
    --url https://github.com/nikever/kubernetes-hello-app \
    --branch main \
    --interval 1m \
    --export \
    > ./clusters/minikube/hello-app/hello-app-source.yaml
```

3. Create a Flux `Kustomization` manifest. This configures Flux to build and apply the kustomize directory located in the *Hello App* repository under the `manifests` folder.

```bash
flux create kustomization hello-app \
  --source=hello-app \
  --path="./manifests" \
  --prune=true \
  --interval=1m \
  --export \
  > ./clusters/minikube/hello-app/hello-app-kustomization.yaml
```

4. Commit and push to deploy in the cluster:

```bash
git add -A && git commit -m "Add Hello App"
git push
```

5. Wait for Flux to reconcile everything:

```bash
watch flux get sources git
watch flux get kustomizations
```

6. Wait for pods to be up and running:

```bash
kubectl get pods -n hello -w
```

7. Test the application:

```bash
curl -H "Host: hello-world.info" $(minikube ip)
```

Expected output:

```bash
Hello, world!
Version: 1.0.0
Hostname: hello-app-5c4957dcc4-l4mqz
```

## 3. Clean up

To clean up:

- Delete the `minikube` cluster:

```bash
cd $REPO_DIR/minikube
make delete
```

- Delete the `flux-fleet` repository.

## Reference

- <https://toolkit.fluxcd.io/get-started/>
- <https://github.com/fluxcd/flux2-multi-tenancy>
