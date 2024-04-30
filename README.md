# Actia
## Create k3d cluster

Create cluster 
```bash
k3d cluster create mycluster --port "80:80@loadbalancer" --port "443:443@loadbalancer"
```

> K3D ARGS: --registry-use, --volume

> K3D Registry: k3d registry create NAME --port PORT

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
## Grafana 
```bash
sudo helm install my-grafana grafana/grafana
```
```bash
sudo kubectl get secret --namespace default my-grafana -o yaml
```
```bash
echo "PASSWORD" | openssl base64 -d ; echo
```

## Adding federate Job 
```bash
sudo KUBE_EDITOR="nano" kubectl edit cm prometheus-server
```
```bash
    - job_name: federate
      scrape_interval: 1m
      honor_labels: true
      metrics_path: 'federate'
      params:
        'match[]':
          - '{job="kubernetes-pods"}'
          - '{job="kubernetes-apiservers"}'
          - '{job="kubernetes-nodes-cadvisor"}'
          - '{job="kubernetes-nodes"}'
          - '{job="kubernetes-service-endpoints"}'
          - '{__name__="cluster_name:.*"}'
      static_configs:
      - targets:
        - '127.0.0.1:9090'
        - '10.0.2.15:31370'
```
```bash
k3d cluster edit cluster-lb --port-add 31370:31370@server:0
```
## Alerts Rules
```bash
  alerting_rules.yml: |-
    groups:
      - name: alert
        rules:
        - alert: InstanceDown
          expr: up == 0
          for: 5s
          labels:
            severity: "critical"
          annotations:
            summary: "Endpoint {{ $labels.instance }} down"
            description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."
        - alert: HostOutOfMemory
          expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
          for: 2m
          labels:
            severity: "warning"
          annotations:
              summary: "Host out of memory (instance {{ $labels.instance }})"
              description: "Node memory is filling up (< 10% left)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: HostUnusualNetworkThroughputIn
          expr: (sum by (instance) (rate(node_network_receive_bytes_total[2m])) / 1024 / 1024 > 100) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: Host unusual network throughput in (instance {{ $labels.instance }})
            description: "Host network interfaces are probably receiving too much data (> 100 MB/s)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: HostUnusualDiskReadRate
          expr: (sum by (instance) (rate(node_disk_read_bytes_total[2m])) / 1024 / 1024 > 50) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: Host unusual disk read rate (instance {{ $labels.instance }})
            description: "Disk is probably reading too much data (> 50 MB/s)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: HostOutOfDiskSpace
          expr: ((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 10 and ON (instance, device, mountpoint) node_filesystem_readonly == 0) * on(instance) group_left (nodename) node_uname>
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Host out of disk space (instance {{ $labels.instance }})
            description: "Disk is almost full (< 10% left)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: HostDiskWillFillIn24Hours
          expr: ((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 10 and ON (instance, device, mountpoint) predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs"}[1h], 24 * 3600) < >
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Host disk will fill in 24 hours (instance {{ $labels.instance }})
            description: "Filesystem is predicted to run out of space within the next 24 hours at current write rate\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: HostHighCpuLoad
          expr: (sum by (instance) (avg by (mode, instance) (rate(node_cpu_seconds_total{mode!="idle"}[2m]))) > 0.8) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: Host high CPU load (instance {{ $labels.instance }})
            description: "CPU load is > 80%\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: HostNetworkInterfaceSaturated
          expr: ((rate(node_network_receive_bytes_total{device!~"^tap.*|^vnet.*|^veth.*|^tun.*"}[1m]) + rate(node_network_transmit_bytes_total{device!~"^tap.*|^vnet.*|^veth.*|^tun.*"}[1m])) / node_networ>
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: Host Network Interface Saturated (instance {{ $labels.instance }})
            description: "The network interface \"{{ $labels.device }}\" on \"{{ $labels.instance }}\" is getting overloaded.\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: HostRequiresReboot
          expr: (node_reboot_required > 0) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
          for: 4h
          labels:
            severity: info
          annotations:
            summary: Host requires reboot (instance {{ $labels.instance }})
            description: "{{ $labels.instance }} requires a reboot.\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
        - alert: os_cluster_memory_high
          expr: sum (container_memory_working_set_bytes{id="/"}) / sum (machine_memory_bytes{}) *100 > 90
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Cluster memory usage exceeding 90%"
            description: "Cluster memory usage exceeding 90% for more than 2 minutes."
        - alert: os_cluster_cpu_high
          expr: sum (rate (container_cpu_usage_seconds_total{id="/"}[1m])) / sum (machine_cpu_cores) * 100 >= 90
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Cluster CPU usage exceeding 90%"
            description: "Cluster CPU usage exceeding 90% for more than 2 minutes."
        - alert: os_cluster_filesystem_high
          expr: sum (container_fs_usage_bytes{device=~"^/dev/sd[a-z][1-9]",id="/"}) / sum (container_fs_limit_bytes{device=~"^/dev/sd[a-z][1-9]",id="/"}) * 100 >= 90
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Cluster filesystem usage exceeding 90%"
            description: "Cluster filesystem usage exceeding 90% for more than 2 minutes."
        - alert: os_node_down
          expr: up{job="kubernetes-nodes"} == 0
          for: 2m
          labels:
            severity: "critical"
          annotations:
            summary: "Instance: {{$labels.instance}} is down."
            description: "Instance: {{$labels.instance}} of job {{ $labels.job }} has been down for more than 2 minutes."
```
## PLAYBOOK

```bash
- hosts: all
  become: yes
  gather_facts: true
  collections:
    - kubernetes.core

  tasks:
  - name: verify ssh connectivity
    wait_for:
      host: "{{ansible_host}}"
      port: 22
      state: started
      timeout: 120

  - name: Update apt package cache
    apt:
      update_cache: yes

  - name: install dependencies
    apt:
      name:
        - curl
        - unzip
      state: present

  - name: Install Docker
    apt:
      name: docker.io
      state: present

  - name: Install Helm
    kubernetes.core.helm:
      name: helm
      state: present

  - name: Install kubectl
    apt:
      name: kubectl
      state: present

  - name: Install k3d
    shell: curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

  - name: create server cluster
    shell: k3d cluster create servercluster --port "80:80@loadbalancer"

  - name: Add prometheus chart
    kubernetes.core.helm_repository:
      name: prometheus
      repo_url: https://prometheus-community.github.io/helm-charts

  - name: Add grafana chart
    kubernetes.core.helm_repository:
      name: grafana
      repo_url: grafana https://grafana.github.io/helm-charts
```


