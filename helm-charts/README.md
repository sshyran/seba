# SEBA Helm Charts

This directory contains Helm charts used to install SEBA components on
a Kubernetes cluster.

To install SEBA:

```
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
helm repo add cord https://charts.opencord.org/master
helm dep update nem
helm dep update seba
helm install nem -n nem
helm install seba -n seba
helm upgrade seba --set voltha.etcd-operator.customResources.createEtcdClusterCRD=true seba
```

After installing the SEBA charts above, the following commands can be used to
wait until all pods have started successfully:

```
tools/wait-for-pods.sh
tools/wait-for-pods.sh voltha
```