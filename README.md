# Gitops workflow demo with Kustomize

## Install k3d (or other local k8s cluster like kind, minikube, ...)

MacOS
```
brew install k3d
```

[Other OS](https://github.com/rancher/k3d)

### Create a k3d cluster with 3 workers
```
k3d create --publish 8080:80 --workers 3

export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
```
### If you need another worker
```
k3d add-node
```

## Install Helm 3

MacOS
```
brew install helm@3
```

[Other OS](https://helm.sh/docs/intro/install/)

## Install Flux

Add the fluxcd repo:

```
helm repo add fluxcd https://charts.fluxcd.io
```

### Install the HelmRelease CRD

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/1.0.1/deploy/crds.yaml
```

### Install Flux on Staging cluster
```
kubectl create ns flux

helm install flux \
--set rbac.create=true \
--set git.url=git@github.com:mfamador/gitops-kustomize-demo.git \
--set git.branch=master \
--set git.path="staging" \
--set git.pollInterval=120s \
--set manifestGeneration=true \
--set syncGarbageCollection.enabled=true \
--namespace flux fluxcd/flux 

fluxctl identity --k8s-fwd-ns flux
```

### Create a deploy key on the github gitops repository with write permissions

    Settings -> Deploy keys -> Add deploy key -> check write permission checkbox

### Install Flux Helm Operator
```
helm upgrade -i helm-operator \
--set git.ssh.secretName=flux-git-deploy \
--set workers=2 \
--set helm.versions=v3 \
--namespace flux fluxcd/helm-operator 
````

## Test

### Echo API
```
curl -H "host:echo.domain.com" http://localhost:8080/
```

### Grafana

Check ingress:
```
curl -H "host:grafana.domain.com" http://localhost:8080/
```

Grafana UI:
```
kubectl port-forward svc/prometheus-operator-grafana 8080:80 -n monitoring
open http://localhost:8080
```


## Find out outdated helm charts

```
helm plugin install https://github.com/fabmation-gmbh/helm-whatup

helm repo update
helm whatup --all-namespaces --ignore-repo # you'll need `--ignore-repo` because the plugin is not handling custom charts correctly
```
