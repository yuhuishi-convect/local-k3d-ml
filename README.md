# Machine learning env

This repo spawns a Machine Learning Environnement that consists of the following components
that can be used to achieve common daily ML tasks.

* JupyterHub - let users obtain jupyter notebook environment in a self-serving manner
* Kubeflow pipeline - to schedule and run background jobs
* Elyra - Jupyter notebook extensions to connect to Kubeflow, to enable users to run notebooks as background jobs, and compose multiple notebooks as a pipeline 
* JuiceFS - Posix like file system to allow users share notebooks / data files between each other
* Minio - S3 compatible object storage to back Kubeflow and JuiceFS
* K3D - Run local kubernetes cluster

# Steps 

## Infra

Start a local k3d cluster.

```
k3d cluster create --config manifests/k3d-default.yaml
```
This creates a cluster with 1 worker. The local port `9090` is forwarded to `80` of the loadbalancer node of the cluster. Later it serves as the entry point to services.




## Kubeflow 

Deploy a kubeflow distribution on the local cluster.

```
# env/platform-agnostic-pns hasn't been publically released, so you will install it from master
export PIPELINE_VERSION=1.4.0
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic-pns?ref=$PIPELINE_VERSION"
```

Create an ingress to expose the kubeflow pipeline service.


## JuiceFS (not working)

Add helm repo of juiceFS

```
helm repo add juicefs-csi-driver https://juicedata.github.io/juicefs-csi-driver/
helm repo update
helm install juicefs-csi-driver juicefs-csi-driver/juicefs-csi-driver -n kube-system -f ./manifests/juiceFS.yaml
```

For the backend and metadata, we reuse the minio and mysql deployments that are shipped with Kubeflow distribution.

Create a database in mysql deployment

```
k exec -it -n kubeflow mysql-f7b9b7dd4-mssv8 -- bash 
```

In the prompt 


```
create database juicefs;
```

## JupyterHub


Add helm repo of jupyterhub

```
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update

helm upgrade --cleanup-on-fail \
  --install jhub-release jupyterhub/jupyterhub \
  --namespace jhub \
  --create-namespace \
  --values manifests/hub-config.yaml
```
Spawns a shared PV for all users to share data and files across JupyterHub

```
kubectl apply -f manifests/shared-storage.yaml -n jhub
```

Add KFP and Minio to the JupyterHub proxy server route table

```
kubectl apply -f manifests/add-route-job -n jhub
```


## Visiting the service

* JupyterHub - `http://dev-box:9090/`
* Minio - `http://dev-box:9090/minio/`
* Kubeflow pipeline - `http://dev-box:9090/pipeline/`

# TODOs

- [ ] GPU support
- [ ] User defined images
- [x] Pre-configured kubeflow pipeline runtime
- [x] Expose kfp endpoint so users can view the status of a pipeline run
- [x] Expose minio endpoint so users can download pipeline artifacts

