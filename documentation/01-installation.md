# 01 - Installation
## Openshift - Online install
### Prometheus/Grafana/AlertManager

Openshift already embed a Prometheus/Grafana stack, but this stack is by default dedicated to the cluster resources. You can enable user workload for this stack, but it can be interesting to deploy your own stack.

You can use the `kube-prometheus-stack` operator to deploy your stack :
- https://github.com/prometheus-community/helm-charts
- https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

Images used are located on `quay.io` mostly :
- Prometheus operator : https://quay.io/repository/prometheus-operator/prometheus-operator?tab=tags
- Prometheus server : https://quay.io/repository/prometheus/prometheus?tab=tags
- Prometheus config reloader : https://quay.io/repository/prometheus-operator/prometheus-config-reloader?tab=tags
- Alertmanager : https://quay.io/repository/prometheus/alertmanager?tab=tags
- Grafana : https://hub.docker.com/r/grafana/grafana
- Grafana sidecar : https://quay.io/repository/kiwigrid/k8s-sidecar?tab=tags

Add helm repo and fetch the chart : 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm fetch --untar prometheus-community/kube-prometheus-stack
```

Before installing the chart with helm, you must be aware that prometheus will be installed through a `Job` and will need the `anyuid` SCC to deploy properly. You will also need to specify this SCC for several `ServiceAccount`.

For the job :

```
oc adm policy add-scc-to-user anyuid -z prometheus-stack-kube-prom-admission
```

For the `deployments` :
```
oc adm policy add-scc-to-user anyuid -z prometheus-stack-kube-prom-operator
oc adm policy add-scc-to-user anyuid -z prometheus-stack-kube-state-metrics
oc adm policy add-scc-to-user anyuid -z prometheus-stack-grafana
oc adm policy add-scc-to-user anyuid -z prometheus-stack-kube-prom-prometheus
oc adm policy add-scc-to-user anyuid -z prometheus-stack-kube-prom-alertmanager
```

for the `statefulsets` :
```
oc adm policy add-scc-to-user anyuid -z prometheus-stack-kube-prom-prometheus
oc adm policy add-scc-to-user anyuid -z prometheus-stack-kube-prom-alertmanager
```

Before installing, you can specify a PVC for prometheus in the `values.yaml` file under the `prometheus.prometheusSpec` section:
```yaml
    ## Prometheus StorageSpec for persistent data
    ## ref: https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/storage.md
    ##
    storageSpec:
    ## Using PersistentVolumeClaim
    ##
      volumeClaimTemplate:
        spec:
          storageClassName: thin
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    #    selector: {}
```

You can also add the following code under `grafana` section to enable grafana persistence :
```yaml
grafana:
  enabled: true
  ## Enable persistence using Persistent Volume Claims
  ## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
  ##
  persistence:
    enabled: true
    type: pvc
    # storageClassName: default
    accessModes:
      - ReadWriteOnce
    size: 10Gi
    # annotations: {}
    finalizers:
      - kubernetes.io/pvc-protection
    # selectorLabels: {}
    # subPath: ""
    # existingClaim:
```

You can also change the `accessMode` from `ReadWriteOnce` to `ReadWriteMany` to be able to process updates properly.

To handle the image version of grafan, you can add the following code in the `grafana` section:

```yaml
grafana:
  enabled: true
  image:
    repository: grafana/grafana
    tag: 8.2.2
    sha: ""
    pullPolicy: IfNotPresent
```

Then install the stack :
```
helm upgrade -i prometheus-stack --namespace monitoring-custom kube-prometheus-stack/
```

Perform a scale down and scale up to create the different pods.

Scale down :
```
oc scale deploy prometheus-stack-grafana prometheus-stack-kube-prom-operator prometheus-stack-kube-state-metrics --replicas=0
oc scale sts alertmanager-prometheus-stack-kube-prom-alertmanager prometheus-prometheus-stack-kube-prom-prometheus --replicas=0
```

Scale up:
```
oc scale deploy prometheus-stack-grafana prometheus-stack-kube-prom-operator prometheus-stack-kube-state-metrics --replicas=1
oc scale sts alertmanager-prometheus-stack-kube-prom-alertmanager prometheus-prometheus-stack-kube-prom-prometheus --replicas=1
```

Your prometheus should be running properly:

```console
[root@workstation ~ ]$ oc get pods
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-stack-kube-prom-alertmanager-0   2/2     Running   0          43m
prometheus-prometheus-stack-kube-prom-prometheus-0       2/2     Running   1          43m
prometheus-stack-grafana-84dbbdbd75-vccd8                2/2     Running   0          63m
prometheus-stack-kube-prom-operator-5f4b6479f-9xgkl      1/1     Running   0          63m
prometheus-stack-kube-state-metrics-6f787f797d-r69zq     1/1     Running   0          63m

[root@workstation ~ ]$ oc get pvc
NAME                                                                                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
prometheus-prometheus-stack-kube-prom-prometheus-db-prometheus-prometheus-stack-kube-prom-prometheus-0   Bound    pvc-b57e0bcf-5f64-42fd-8995-e14e36bd93bc   50Gi       RWX            thin                     12m
prometheus-stack-grafana                                                                                 Bound    pvc-b82eb2d1-0c6c-4701-b8da-8d9be6ad3a23   10Gi       RWX            trident-storage-delete   12m
```

### Blackbox exporter

You can use the `prometheus-blackbox-exporter` chart from the prometheus community to deploy blackbox
- https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-blackbox-exporter

Add helm repo and fetch the chart : 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm fetch --untar prometheus-community/prometheus-blackbox-exporter
```

The blackbox exporter pod need the SCC `anyuid` to be properly deployed :

```
oc adm policy add-scc-to-user anyuid -z blackbox-prometheus-blackbox-exporter
```

If you need to set the `allowIcmp` section to `true`, you will need `NET_RAW` capability for your pod. For this purpose, you can create your own SCC or allow `privileged` SCC to the ServiceAccount.


### Values example

You can find a `values.yaml` file completly configured for `kube-prometheus-stack` chart grafana and ldap with latest version of images for helm chart version `19.2.2` here [values yaml file](../resources/helm-chart/values/values.yaml)