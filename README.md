# keda-demo

* [keda-demo](#keda-demo)
  * [Setup](#setup)
    * [Fetch the Helm Repositories](#fetch-the-helm-repositories)
    * [Install KEDA](#install-keda)
    * [Install the Metrics Exporter Workloads (PodInfo)](#install-the-metrics-exporter-workloads-podinfo)
    * [Install a Deployment with 1 Replica](#install-a-deployment-with-1-replica)
  * [Demos](#demos)
    * [Autoscaling a Deployment with a Prometheus Metrics](#autoscaling-a-deployment-with-a-prometheus-metrics)
      * [Prerequisites](#prerequisites)
      * [Steps](#steps)
    * [Autoscaling a Deployment with a Redis List](#autoscaling-a-deployment-with-a-redis-list)
      * [Steps](#steps-1)
    * [Autoscaling Jobs with a Redis List](#autoscaling-jobs-with-a-redis-list)
      * [Steps](#steps-2)

## Setup

These are the steps required for setting up and running the demos.

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

## Demos

### Autoscaling a Deployment with a Prometheus Metrics

#### Prerequisites

* Have the prometheus-operator installed and configured to look for `ServiceMonitor`s in the `keda-demo` namespace. You can use the [community helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) to install it
* Optional, have [Ambassador's Telepresence v2](https://www.getambassador.io/docs/telepresence/latest/install/) installed

#### Steps

1. Update the `serverAddress` in the `examples/keda/prom-scaledobject.yaml` file.
    * If you want to access it interally, you can use the format `http://SVC_NAME.NAMESPACE.svc.cluster.local:9090`. If your prometheus instance is served on a subpath you must attach the subpath after the port number (e.g. `http://SVC_NAME.NAMESPACE.svc.cluster.local:9090/prometheus`)
2. Deploy the `ScaledObject` with the prometheus trigger

    ```
    kubectl apply -f examples/keda/prom-scaledobject.yaml
    ```

3. Check the scaled object is ready with `kubectl get scaledobject -n keda-demo`
4. If it is not ready, check the logs of the keda pod (`keda logs POD_NAME -n keda-demo`). It'll most likely be the server address being incorrect.
5. Check and examine the HPA object created by KEDA with `kubectl get hpa -n keda-demo` and `kubectl describe hpa -n keda-demo`.
6. Open another terminal and watch the pods in the `keda-demo` namespace with `kubectl get pods -n keda-demo -w`.
7. Open another terminal and watch the deployments in the `keda-demo` namespace with `kubectl get deployments -n keda-demo -w`.
8. Now we want to trigger the autoscaling. If you have telepresence, repeatedly run curl commands against the PodInfo service with `podinfo.keda-demo:9898/metrics`.
    * If you don't have telepresence you can exec into another pod with curl installed and run `curl podinfo.keda-demo.svc.cluster.local:9898/metrics` from within the context of that pod,
9. As you run the curl commands, you'll eventually notice the number of replicas of the consumer workload increase in the other terminals.
10. Stop running the curl commands.
11. Wait until the `Deployment` scales down back to the minimum number of replicas
12. Clean up the resources.

    ```
    kubectl delete -f examples/keda/prom-scaledobject.yaml
    ```

### Autoscaling a Deployment with a Redis List

#### Steps

1. Deploy the redis `Deployment` and `Service`
    ```
    kubectl apply -f examples/deployments/redis.yaml
    kubectl apply -f examples/deployments/redis-svc.yaml
    ```
2. Confirm the redis pod is running with `kubectl get pods -n keda-demo`
3. Confirm the redis service has successfully found the redis pod by checking an endpoint exists with `kubectl get endpoints -n keda-demo`
4. Deploy the `ScaledObject` with the redis trigger

    ```
    kubectl apply -f examples/keda/redis-scaledobject.yaml
    ```

5. Check the scaled object is ready with `kubectl get scaledobject -n keda-demo`
6. If it is not ready, check the logs of the keda pod (`keda logs POD_NAME -n keda-demo`). It'll most likely be the server address being incorrect.
7. Check and examine the HPA object created by KEDA with `kubectl get hpa -n keda-demo` and `kubectl describe hpa -n keda-demo`.
8. Open another terminal and watch the pods in the `keda-demo` namespace with `kubectl get pods -n keda-demo -w`
9. Open another terminal and watch the deployments in the `keda-demo` namespace with `kubectl get deployments -n keda-demo -w`
10. Now we want to trigger the autoscaling. Exec into the redis pod with `kubectl exec redis-(RANDOM_STRING) -it -n keda-demo -- redis-cli`
11. Check the length of the list stored in the `mylist` key with `LLEN mylist`
12. Add to the list with `LLPUSH mylist "string"` until the length of the list is above the threshold.
13. One additional replica should have been created.
14. Continuing adding items to the list until the threshold is `(threshold x 2) + 1)`.
15. Another replica should be created.
16. Remove items from the list with `LPOP mylist` until the length of the list is below the threshold.
17. Wait until the `Deployment` scales down back to the minimum number of replicas.
18. Clean up the resources.

    ```
      kubectl delete -f examples/deployments/redis.yaml
      kubectl delete -f examples/deployments/redis-svc.yaml
      kubectl delete -f examples/keda/redis-scaledobject.yaml
    ```

### Autoscaling Jobs with a Redis List

#### Steps

1. Deploy the redis `Deployment` and `Service`
    ```
    kubectl apply -f examples/deployments/redis.yaml
    kubectl apply -f examples/deployments/redis-svc.yaml
    ```
2. Confirm the redis pod is running with `kubectl get pods -n keda-demo`
3. Confirm the redis service has successfully found the redis pod by checking an endpoint exists with `kubectl get endpoints -n keda-demo`
4. Deploy the `ScaledJob` with the redis trigger.

    ```
    kubectl apply -f examples/keda/redis-scaledjob.yaml
    ```

5. Check the scaled object is ready with `kubectl get scaledobject -n keda-demo`
6. If it is not ready, check the logs of the keda pod (`keda logs POD_NAME -n keda-demo`). It'll most likely be the server address being incorrect.
7. Open another terminal and watch the pods in the `keda-demo` namespace with `kubectl get pods -n keda-demo -w`
8. Open another terminal and watch the jobs in the `keda-demo` namespace with `kubectl get jobs -n keda-demo -w`
9.  Now we want to trigger the autoscaling. Exec into the redis pod with `kubectl exec redis-(RANDOM_STRING) -it -n keda-demo -- redis-cli`
10. Check the length of the list stored in the `myotherlist` key with `LLEN myotherlist`
11. Add to the list with `LLPUSH myotherlist "myotherlist"` until the length of the list is above the threshold.
12. One additional replica should have been created.
13. Wait and observe as more jobs continously get created.
14. Remove the other from the list with `LPOP myotherlist`
15. Wait and observe as no more jobs are created.
16. Clean up the resources.

    ```
      kubectl delete -f examples/deployments/redis.yaml
      kubectl delete -f examples/deployments/redis-svc.yaml
      kubectl delete -f examples/keda/redis-scaledjob.yaml
    ```
