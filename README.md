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
