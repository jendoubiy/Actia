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
