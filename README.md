# Actia
## Create k3d cluster

Create cluster 
```bash
k3d cluster create mycluster
```

Get the new cluster’s connection 
```bash
k3d kubeconfig merge mycluster --kubeconfig-switch-context
```

Connect Kubectl to ConfigFile 
```bash
export KUBECONFIG=/root/.config/k3d/kubeconfig-mycluster.yaml
```

**************
Create a nginx deployment
```bash
sudo kubectl create deployment nginx --image=nginx:alpine

```

create a Clusterip service for it 
```bash
sudo kubectl create service clusterip nginx --tcp=80:80
```




## ingress
Create an ingress object 
```bash
sudo kubectl apply -f ./ingress.yaml
```
ingress.yaml
```python
ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - http:
        paths:
          - path: /nginx
            pathType: ImplementationSpecific
            backend:
              service:
                name: nginx
                port:
                  number: 80
```
# Remote cluster

Create Remote cluster with persistence Volume 

## Creation

```bash
sudo k3d cluster create remotecluster
```
```bash
export KUBECONFIG=/root/.config/k3d/kubeconfig-remotecluster.yaml
```
## Volume
create Volume.yaml
```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/volume"
```
```bash
sudo kubectl apply -f volume.yaml
```

Verification !
```bash
sudo kubectl get pv
```
Adding helm repo Config to cluster
```bash
export HELM_REPOSITORY_CONFIG=/root/.config/k3d/kubeconfig-remotecluster.yaml
```
installation prometheus using helm 
```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.persistence.enabled=true \
  --set server.persistence.existingClaim=prometheus \
  --set server.resources.requests.memory="2Gi" \
  --set server.resources.requests.cpu="1"
```
Expose prometheus server 9090
```bash
sudo kubectl port-forward prometheus-server-59bb469b7b-6btv9 9090
```
