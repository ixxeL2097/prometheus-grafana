# External scrape

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: workload
  name: nfs-centralus-001
  labels:
    workload.stateful: nfs-centralus-001
spec:
  type: ExternalName
  externalName: nfs-centralus-001.c.saas-workload-io.internal
  selector:
    workload.stateful: nfs-centralus-001
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    ops.workload.io/component: nfs-centralus-001
    ops.workload.io/category: infrastructure
  name: nfs-centralus-001
  namespace: workload
spec:
  endpoints:
  - path: /metrics
    interval: 15s
    targetPort: 9100
    scheme: http
    relabelings:
      - sourceLabels: [__address__]
        targetLabel: __address__
        regex: (.*)
        replacement: "$FQDN:$PORT"
        action: replace
  jobLabel: ops.workload.io/nfs-centralus-001
  namespaceSelector:
    matchNames:
    - workload
  selector:
    matchExpressions:
      - key: workload.stateful
        operator: In
        values: ["nfs-centralus-001"]
```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: nfs-centralus-001
  namespace: workload
  labels:
    workload.stateful: nfs-centralus-001
subsets:
- addresses:
  - ip: 1.2.3.4
  - ip: 1.2.3.5
  ports:
  - name: metrics
    port: 9100
    protocol: TCP
```

A quick note on this, as I was struggling with finding the same config:

You don't even need to specify the real IP of your destination FQDN in the Endpoint, it can be just any IP, because by relabeling __address__ you're instructing Prometheus to actually scrape what is specified in the __address__ label.

This would be usually populated by the IP address that is defined in the Endpoint, but we're replacing it with a whole different address here, so the actual IP in the Endpoint doesn't even matter to Prometheus itself anymore.