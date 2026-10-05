# Kubernetes Demo - Complete Lab Notes

This folder contains the Kubernetes manifests used during the lab to understand how multiple resources interact in a cluster. The demo progressed from a basic NGINX application to a more advanced setup with a second app, ingress routing, RBAC, and network policies.

The purpose of this lab was to practice:
- creating Kubernetes manifests
- applying them to a local Kind cluster
- investigating scheduling issues
- checking pod health and service reachability
- understanding ConfigMaps, Secrets, PV/PVC, Services, Ingress, RBAC, and Network Policies

---

## Files currently in the Demo folder

| File | Kind | Purpose |
| --- | --- | --- |
| `configmap.yaml` | ConfigMap | Stores the custom homepage HTML for the main NGINX app. |
| `deployment.yaml` | Deployment | Runs the main NGINX app with health probes, resource limits, and PVC/ConfigMap mounts. |
| `external-service.yaml` | Service | Exposes the main app using a NodePort. |
| `pv.yaml` | PersistentVolume | Defines a hostPath-based persistent volume. |
| `pvc.yaml` | PersistentVolumeClaim | Requests storage for the main application. |
| `secret.yaml` | Secret | Stores demo credentials as environment variables. |
| `app2.yaml` | Deployment + Service | Deploys a second sample app (`app2`) using `hashicorp/http-echo`. |
| `ingress.yaml` | Ingress | Routes HTTP traffic to the main app and the second app. |
| `network-policy.yaml` | NetworkPolicy | Allows only pods labeled `app: nginx` to reach the `app2` container. |
| `role.yaml` | Role | Gives read-only access to pods in the namespace. |
| `rolebinding.yaml` | RoleBinding | Binds the Role to a ServiceAccount. |
| `clusterrole.yaml` | ClusterRole | Gives read-only cluster-level access to pods. |
| `clusterrolebinding.yaml` | ClusterRoleBinding | Binds the cluster role to a ServiceAccount. |
| `readme.md` | Documentation | Lab notes, concepts, and commands used. |

---

## Core concepts behind the manifests

### 1) ConfigMap

The main app serves custom HTML content, which is stored in the ConfigMap.

```yaml
configMap:
  name: nginx-config
```

This is used to avoid hardcoding site content inside the image and makes the app easier to manage and update.

### 2) Secret

The Secret stores values for the environment variables used by the container.

```yaml
env:
  - name: APP_USERNAME
    valueFrom:
      secretKeyRef:
        name: nginx-secret
        key: USERNAME
```

This demonstrates Kubernetes Secret injection. In this lab, the stored values were replaced with placeholder text before publishing to GitHub.

### 3) PersistentVolume and PersistentVolumeClaim

The app uses a PV/PVC pair to persist data on the node filesystem.

```yaml
hostPath:
  path: /data/nginx
```

This allows the NGINX content to survive a pod restart in a local lab environment.

Key idea:
- PV = the actual storage resource
- PVC = the storage request made by an app

### 4) Deployment and health probes

The main deployment includes:
- `replicas: 1`
- `nodeSelector: app-node=web`
- `startupProbe`
- `readinessProbe`
- `livenessProbe`
- resource requests and limits

This is used so Kubernetes can determine when the app is ready and when it needs to be restarted.

### 5) Service and NodePort access

The Service exposes the NGINX app through a NodePort.

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

This allows access from outside the cluster in a local setup.

### 6) Additional app and ingress flow

The lab expanded to a second application:
- `app2.yaml` creates a sample app using `hashicorp/http-echo`
- `ingress.yaml` routes requests to `nginx-external` and `app2-service`

This introduced the concept of application routing and layered traffic flow in Kubernetes.

### 7) ServiceAccount and RBAC (Role and ClusterRole)

A ServiceAccount is the identity used by applications, automation, and internal workloads running inside the cluster.

Example workflow used in this lab:

```bash
kubectl create serviceaccount dev-reader
kubectl create token dev-reader
```

This creates a service account named `dev-reader` and generates a token that can be used for authentication.

The RBAC manifests then define which identities can read pod information:
- `role.yaml` and `rolebinding.yaml` apply within a namespace
- `clusterrole.yaml` and `clusterrolebinding.yaml` apply cluster-wide

`RoleBinding` connects a Role to a ServiceAccount. In other words, the ServiceAccount gets permissions described by the Role when the binding exists.

Example:

```yaml
subjects:
  - kind: ServiceAccount
    name: dev-reader
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This means: the `dev-reader` ServiceAccount is allowed to perform the actions defined in the `pod-reader` Role.

### 8) NetworkPolicy

The NetworkPolicy allows only the NGINX pod to communicate with `app2`.

```yaml
podSelector:
  matchLabels:
    app: app2
```

This shows how Kubernetes can restrict which workloads can access a pod, which is an important security pattern.

---

## Commands used during the lab

The following are the commands used while creating, applying, investigating, and testing the manifests.

### Create and verify the Kind cluster

```bash
kind create cluster --name multinode --config kind-multinode.yml
kubectl config current-context
kubectl get nodes -o wide
kubectl cluster-info
```

### Label the worker node for the main app

```bash
kubectl get nodes
kubectl label node <worker-node-name> app-node=web
```

### Apply the main manifests

```bash
kubectl apply -f .
```

or selectively:

```bash
kubectl apply -f configmap.yaml
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f external-service.yaml
```

### Apply the extra advanced manifests

```bash
kubectl apply -f app2.yaml
kubectl apply -f ingress.yaml
kubectl apply -f network-policy.yaml
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml
```

### Check resource state

```bash
kubectl get pods
kubectl get deployment
kubectl get svc
kubectl get ingress
kubectl get networkpolicy
kubectl get pv,pvc
kubectl get role,rolebinding
kubectl get clusterrole,clusterrolebinding
```

### Inspect details when things do not work

```bash
kubectl describe pod nginx-app
kubectl describe pod app2
kubectl describe svc nginx-external
kubectl describe svc app2-service
kubectl describe ingress nginx-ingress
kubectl describe pv nginx-pv
kubectl describe pvc nginx-pvc
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Check logs

```bash
kubectl logs deployment/nginx-app
kubectl logs deployment/app2
```

### Access the app locally

```bash
kubectl port-forward service/nginx-external 8080:80
```

Then open:

```text
http://localhost:8080
```

To validate the response:

```bash
curl http://localhost:8080
```

### Investigate RBAC and permissions

```bash
kubectl create serviceaccount dev-reader
kubectl create token dev-reader
kubectl auth can-i get pods --as=system:serviceaccount:default:dev-reader
kubectl auth can-i list pods --as=system:serviceaccount:default:dev-reader
```

This helps confirm whether the Role or ClusterRole grants access to a ServiceAccount.

### Set the current context with a ServiceAccount token

When you want to switch the active kube context to use a token-based identity, the usual pattern is:

```bash
TOKEN=$(kubectl create token dev-reader)
kubectl config set-credentials dev-reader --token="$TOKEN"
kubectl config set-context --current --user=dev-reader
kubectl config current-context
```

This creates a token for the `dev-reader` ServiceAccount and saves it as the active kubeconfig user. Then `kubectl` commands run using that token identity. This is useful when testing RBAC behavior without using the default admin identity.

---

## Troubleshooting lessons from this lab

### 1) Pending pod because of node selector

The main deployment used:

```yaml
nodeSelector:
  app-node: web
```

If the node is not labeled, the pod stays Pending. The fix is:

```bash
kubectl label node <worker-node-name> app-node=web
```

### 2) Pending PVC because of storage binding

If the PVC does not bind to the PV, the app waits for storage. In this lab, the cause was usually mismatch between the storage class or the PV/PVC configuration.

Check:

```bash
kubectl get pv,pvc
kubectl describe pvc nginx-pvc
kubectl describe pv nginx-pv
```

### 3) Ingress not routing traffic

Ingress needs a working ingress controller in the cluster. If the rule is created but not working, investigate:

```bash
kubectl get ingress
kubectl describe ingress nginx-ingress
kubectl get pods -n ingress-nginx
```

### 4) Network policy blocks communication

The NetworkPolicy restricts traffic to the `app2` pod. If access fails, inspect the policy and the pod labels.

```bash
kubectl get networkpolicy
kubectl describe networkpolicy allow-nginx-to-app2
kubectl get pods --show-labels
```

### 5) RBAC denial

If a ServiceAccount cannot read pods, investigate the Role/RoleBinding or ClusterRole/ClusterRoleBinding configuration.

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:dev-reader
kubectl describe rolebinding pod-reader-binding
kubectl describe clusterrolebinding pod-cluster-reader-binding
```

---

## Cleanup commands

When the demo is complete:

```bash
kubectl delete -f .
kind delete cluster --name multinode
```

This removes all resources created during the lab and deletes the Kind cluster itself.

---

## Final takeaway

This Demo folder is a full learning path for Kubernetes basics and intermediate concepts. It demonstrated:
- configuration management with ConfigMaps and Secrets
- persistence with PV and PVC
- pod lifecycle and health checks with Deployment probes
- service exposure with NodePort and Ingress
- security with RBAC and NetworkPolicy
- troubleshooting using `kubectl` and Kubernetes events

This lab is a strong foundation before moving to production patterns such as Helm charts, ingress controllers, storage classes, and advanced security best practices.