# keda-demo

## Install KEDA

```
helm repo add kedacore https://kedacore.github.io/charts
```

Update Helm repo

```
helm repo update
```

Install keda Helm chart

```
kubectl create namespace keda-demo
helm install keda kedacore/keda --namespace keda-demo --version 2.3.2
```
