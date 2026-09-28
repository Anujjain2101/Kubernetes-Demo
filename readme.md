# NGINX Kubernetes Demo

This folder contains Kubernetes manifests for a single-replica NGINX application intended for a local Kind cluster. The app serves a custom HTML page, mounts persistent storage, reads Secret values into environment variables, and is exposed through a NodePort Service.

## Files

| File | Resource | Purpose |
| --- | --- | --- |
| `configmap.yaml` | ConfigMap `nginx-config` | Supplies the `index.html` page mounted into NGINX. |
| `secret.yaml` | Secret `nginx-secret` | Supplies `USERNAME` and `PASSWORD` values to the container. |
| `pv.yaml` | PersistentVolume `nginx-pv` | Provides 1 GiB of `hostPath` storage at `/data/nginx` with a `Retain` reclaim policy. |
| `pvc.yaml` | PersistentVolumeClaim `nginx-pvc` | Requests 1 GiB of `ReadWriteOnce` storage. |
| `deployment.yaml` | Deployment `nginx-app` | Runs one `nginx:latest` pod, mounts the PVC and ConfigMap, and checks readiness over HTTP. |
| `external-service.yaml` | NodePort Service `nginx-external` | Routes port 80 to the app and exposes NodePort 30080 inside the Kind cluster. |

## Prerequisites

- Docker Desktop running
- Kind and `kubectl` installed
- A running Kubernetes cluster and a configured `kubectl` context

The sibling `k8s/kind-multinode.yml` and `k8s/readme.md` describe creating the local three-node cluster named `multinode`.

## Deploy

Run these commands from the `k8s/Demo` directory. The Deployment requires a node labeled `app-node=web`; Kind does not add this label automatically. Choose a worker node from `kubectl get nodes` and label it before applying the manifests:

```powershell
kubectl config current-context
kubectl get nodes
kubectl label node <worker-node-name> app-node=web
kubectl apply -f .
```

Check the rollout and storage binding:

```powershell
kubectl rollout status deployment/nginx-app
kubectl get pods,services,pv,pvc
```

The PVC must bind to the PV before the pod can become ready. The PV declares storage class `standard`, while the PVC leaves its storage class unspecified. If the PVC remains `Pending`, explicitly set `storageClassName: standard` in `pvc.yaml` (or make the PV and PVC storage-class settings match your cluster), then apply the updated manifest.

## Test the page

Port-forward the Service and open `http://localhost:8080` in a browser:

```powershell
kubectl port-forward service/nginx-external 8080:80
```

The page content comes from the `index.html` entry in `configmap.yaml`. The Deployment's readiness probe checks `/` on port 80.

## Configuration and security notes

- `deployment.yaml` selects nodes with the label `app-node=web`; without a matching node, the pod remains unscheduled.
- The Deployment reads the Secret keys into `APP_USERNAME` and `APP_PASSWORD`. The stock NGINX image does not use these variables for authentication; they are only present in the container environment.
- `secret.yaml` contains placeholder values only. Replace them before deploying, or create the Secret outside the repository. Never commit real credentials.
- `hostPath` storage is local to a Kind node and is suitable for this local demo, not portable or highly available production storage.
- The NodePort is 30080 inside the cluster. Port-forwarding is used above to access the app from the host without additional Kind port mappings.

## Remove the resources

Run from this directory:

```powershell
kubectl delete -f .
```

Because the PV has reclaim policy `Retain`, deleting the claim does not automatically erase the data stored at `/data/nginx` on the Kind node. Remove the Kind cluster separately when you no longer need it:

```powershell
kind delete cluster --name multinode
```