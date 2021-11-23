# 04 - Configuration Blackbox exporter
## Blackbox
### Modules

Modules define how blackbox exporter is going to query the endpoint, therefore one needs to be created for each request type under the `config.modules` section of the chart.

Simple standard HTTP module probe :
```yaml
config:
  modules:
    http_2xx:
      prober: http
      timeout: 5s
      http:
        valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
        follow_redirects: true
        preferred_ip_protocol: "ip4"
```

check this link for further information :
- https://lyz-code.github.io/blue-book/devops/prometheus/blackbox_exporter/#blackbox-exporter-probes

