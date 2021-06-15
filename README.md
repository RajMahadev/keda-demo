# keda-demo

## Prerequisites

* Have the prometheus-operator installed and configured to look for `ServiceMonitor`s in the `keda-demo` namespace. You can use the community 

## Setup

These are the steps required for setting up and running the demo.

### Fetch the Helm Repositories

Get the required helm repositories with the commands below:


```
helm repo add kedacore https://kedacore.github.io/charts
helm repo add podinfo https://stefanprodan.github.io/podinfo
```

Update the Helm Repos

```
helm repo update
```

### Install KEDA

Create a namespace and install the Keda Helm chart in it

```
kubectl create namespace keda-demo
helm install keda kedacore/keda --namespace keda-demo --version 2.3.2
```

### Install the Metrics Exporter Workloads (PodInfo)

Install the PodInfo helm chart in the same namespace as Keda. You want to use the `--set` flag to enable the `ServiceMonitor`. If the prometheus-operator is configured correctly then the `ServiceMonitor` will be detected and Prometheus will be configured to scape metrics from the PodInfo workload(s).

```
helm install podinfo --namespace keda-demo podinfo/podinfo --version 5.2.1 --set serviceMonitor.enabled=true
```

After installation, go to the `/prometheus/targets` endpoint for you prometheus instance and confirm that the PodInfo workload is a target that is successfully being scraped.

### Install a Deployment with 1 Replica

Deploy a `Deployment` object with 1 replica to the `keda-demo` namespace.

```
kubectl apply -n keda-demo -f examples/deployments/1-replica.yaml
```