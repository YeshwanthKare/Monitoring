## Monitoring Cluster

- kube-state-metrics

### Prometheus

- TimeSeries Database
  - Scrape Data
  - Query Server (PromQL)
  - Graphs
  - Data Source

After this we have Visualization, which can be done by using Garafana

### Grafana

- Visualizations
- Data Sources

### Installing Kind Cluster

Install Kind

```
#!/bin/bash

# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo cp ./kind /usr/local/bin/kind
rm -rf kind
```

Check kind version

```
kind --version
```

Bring up a multi node cluster
Create a config file (`config.yml`) with the below content:

```
# 4 node (3 workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.34.0
- role: worker
  image: kindest/node:v1.34.0
- role: worker
  image: kindest/node:v1.34.0
- role: worker
  image: kindest/node:v1.34.0
```

Start the 4 node cluster on this host:

```
kind create cluster --config=config.yml
```

Check the cluster status:

Using kind based command:

```
kind get clusters
```

Using kubectl based command. If you do not have kubectl installed, please refer to install kubectl linux to install it:

```
sudo snap install kubectl --classic

kubectl cluster-info
```

Check the status of each node on the cluster:

```
kubectl get nodes
```

Create a Namespace monitoring

```
kubectl create ns monitoring
```

> Install Helm

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

> Add the Prometheus Helm repo

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update
```

> Install prometheus into the monitoring namespace

```
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --set prometheus.service.nodePort=30000 --set grafana.service.nodePort=31000 --set grafana.service.type=NodePort --set prometheus.service.type=NodePort

# Output

NAME: prometheus-stack
LAST DEPLOYED: Sun Sep 21 09:14:36 2025
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=prometheus-stack"

Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus-stack" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

> Setting the current context to monitoring

```
kubectl config set-context --current --namespace=monitoring
```
