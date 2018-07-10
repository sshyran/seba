# SEBA Helm Charts

This directory contains Helm charts used to install SEBA components on
a Kubernetes cluster.

To install SEBA:

```
helm dep update nem
helm dep update seba
helm install nem -n nem
helm install seba -n seba
helm upgrade seba --set voltha.etcd-operator.customResources.createEtcdClusterCRD=true seba
```

The following commands will wait until all SEBA pods have started successfully:

```
tools/wait-for-pods.sh
tools/wait-for-pods.sh voltha
```