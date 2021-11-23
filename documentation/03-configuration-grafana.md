# 03 - Configuration Grafana
## Grafana
### Dashboards : manual

To import dashboards into Grafana, you can do it via Helm chart `values.yaml` adding the following code into `grafana` section :

```yaml
grafana:
  enabled: true

  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default

  ## Reference to external ConfigMap per provider. Use provider name as key and ConfigMap name as value.
  ## A provider dashboards must be defined either by external ConfigMaps or in values.yaml, not in both.
  ## ConfigMap data example:
  ##
  ## data:
  ##   example-dashboard.json: |
  ##     RAW_JSON
  ##
  dashboardsConfigMaps:
    default: "grafana-global-dashboards"
```

The dashboard provider is mandatory to reference to a configMap. To create a configMap from json file, use following command :

```
oc create cm blackbox-grafana-dashboard --from-file=blackbox-dashboard.json
```
Or create a configMap with multiple dashboards from a directory:
```
oc create cm grafana-global-dashboards --from-file=shared-services-dashboards/
```

The dashboardProvider is referenced in the `prometheus-stack-grafana` configMap :
```console
[root@workstation ~ ]$ oc get cm prometheus-stack-grafana -oyaml | oc neat
apiVersion: v1
data:
  dashboardproviders.yaml: |
    apiVersion: 1
    providers:
    - disableDeletion: false
      editable: true
      folder: ""
      name: default
      options:
        path: /var/lib/grafana/dashboards/default
      orgId: 1
      type: file
  grafana.ini: |
    [analytics]
    check_for_updates = true
    [grafana_net]
    url = https://grafana.net
    [log]
    mode = console
    [paths]
    data = /var/lib/grafana/data
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus-stack
    meta.helm.sh/release-namespace: monitoring-custom
  labels:
    app.kubernetes.io/instance: prometheus-stack
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 7.5.3
    helm.sh/chart: grafana-6.8.1
  name: prometheus-stack-grafana
  namespace: monitoring-custom
```

Then, in grafana deployment, the dashboardProvider **path** is mounted as following (example for `default` dashboardProvider):

```yaml
        volumeMounts:
        - mountPath: /var/lib/grafana/dashboards/default
          name: dashboards-default
```

And finally, configMap is mounted on that volumeMount :
```yaml
      volumes:
      - configMap:
          defaultMode: 420
          name: grafana-global-dashboards
        name: dashboards-default
```

### Dashboards : sidecar

However, the best way to do it is to use sidecar container to watch for configMap to be loaded as a dashboard. The sidecar container will add resources with the correct label as a dashboard into Grafana UI.

The following section is already available in `values.yaml` :

```yaml
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard

      ## Annotations for Grafana dashboard configmaps
      ##
      annotations: {}
```

You can modify it as following :

```yaml
  sidecar:
    image:
      repository: quay.io/kiwigrid/k8s-sidecar
      tag: 1.14.2
      sha: ""
    imagePullPolicy: IfNotPresent
    resources: {}
  #   limits:
  #     cpu: 100m
  #     memory: 100Mi
  #   requests:
  #     cpu: 50m
  #     memory: 50Mi
    # skipTlsVerify Set to true to skip tls verification for kube api calls
    # skipTlsVerify: true
    enableUniqueFilenames: false
    dashboards:
      enabled: true
      SCProvider: true
      # label that the configmaps with dashboards are marked with
      label: grafana_dashboard
      # value of label that the configmaps with dashboards are set to
      labelValue: 1
      # folder in the pod that should hold the collected dashboards (unless `defaultFolderName` is set)
      folder: /tmp/dashboards
      # The default folder name, it will create a subfolder under the `folder` and put dashboards in there instead
      defaultFolderName: null
      # Namespaces list. If specified, the sidecar will search for config-maps/secrets inside these namespaces.
      # Otherwise the namespace in which the sidecar is running will be used.
      # It's also possible to specify ALL to search in all namespaces.
      searchNamespace: ALL
      # search in configmap, secret or both
      resource: both
      # If specified, the sidecar will look for annotation with this name to create folder and put graph here.
      # You can use this parameter together with `provider.foldersFromFilesStructure`to annotate configmaps and create folder structure.
      folderAnnotation: grafana_folder
      # provider configuration that lets grafana manage the dashboards
      provider:
        # name of the provider, should be unique
        name: sidecarProvider
        # orgid as configured in grafana
        orgid: 1
        # folder in which the dashboards should be imported in grafana
        folder: ''
        # type of the provider
        type: file
        # disableDelete to activate a import-only behaviour
        disableDelete: false
        # allow updating provisioned dashboards from the UI
        allowUiUpdates: true
        # allow Grafana to replicate dashboard structure from filesystem
        foldersFromFilesStructure: true
```

In this example, all configMaps with the label `grafana_dashboard: "1"` will be added as dashboard into Grafana. First create your configMap from json file and then add a label to the configMap :
```
oc create cm grafana-argocd-dashboard --from-file=configMap/argocd-dashboard.json
oc label cm grafana-argocd-dashboard grafana_dashboard=1
```

The option `folderAnnotation` allow you to annotate configMaps to store your dashboards in specific folders :
```
oc annotate cm grafana-argocd-dashboard grafana_folder=/tmp/dashboards/shared-services
```

The result is the following :
```yaml
kind: ConfigMap
metadata:
  annotations:
    grafana_folder: /tmp/dashboards/shared-services
  labels:
    grafana_dashboard: "1"
```

For each folder you want to create, you need a `dashboardProviders`. For example, if you want default, shared-services and apps, you need the following lines added to your `values.yaml` :

```yaml
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'CI/CD'
        orgId: 1
        folder: 'CI/CD'
        type: file
        disableDeletion: true
        editable: true
        allowUiUpdates: true
        options:
          path: /tmp/dashboards/ci-cd
      - name: 'Shared Services'
        orgId: 1
        folder: 'Shared Services'
        type: file
        disableDeletion: true
        editable: true
        allowUiUpdates: true
        options:
          path: /tmp/dashboards/shared-services
      - name: 'Apps'
        orgId: 1
        folder: 'Apps'
        type: file
        disableDeletion: true
        editable: true
        allowUiUpdates: true
        options:
          path: /tmp/dashboards/apps
```

If you want the default dashboards, you can use the variable `defaultDashboardsEnabled: true`. They will be stored in `General` directory in Grafana UI.

### LDAP

To activate LDAP, you must have the following code in the `grafana` section:

```yaml
grafana:
  enabled: true

  grafana.ini:
  ## LDAP Authentication can be enabled with the following values on grafana.ini
  ## NOTE: Grafana will fail to start if the value for ldap.toml is invalid
    auth.ldap:
      enabled: true
      allow_sign_up: true
      config_file: /etc/grafana/ldap.toml
```

Then you can configure your LDAP server again under the `grafana` section. Add the following lines and customize it according to your needs:

```yaml
grafana:
  enabled: true

  ldap:
    enabled: true
    # `existingSecret` is a reference to an existing secret containing the ldap configuration
    # for Grafana in a key `ldap-toml`.
    existingSecret: ""
    # `config` is the content of `ldap.toml` that will be stored in the created secret

    config: |-
      verbose_logging = true

      [[servers]]
      host = "10.194.216.85"
      port = 389
      use_ssl = false
      start_tls = false
      ssl_skip_verify = false
      bind_dn = "cn=ldap_grafana,cn=Users,dc=devibm,dc=local"
      bind_password = 'Passw0rd'
      search_filter = "(|(sAMAccountName=%s)(userPrincipalName=%s))"
      search_base_dns = ["dc=devibm,dc=local"]
      [servers.attributes]
      name = "givenName"
      surname = "sn"
      username = "sAMAccountName"
      member_of = "memberOf"
      email =  "email"
```

If you want to setup group sync, then add `servers.group_mappings` :
```yaml
      [[servers.group_mappings]]
      group_dn = "cn=grafana_admins,ou=grafana,ou=Shared-services,dc=devibm,dc=local"
      org_role = "Admin"
      grafana_admin = true
      [[servers.group_mappings]]
      group_dn = "cn=grafana_editors,ou=grafana,ou=Shared-services,dc=devibm,dc=local"
      org_role = "Editor"
      [[servers.group_mappings]]
      group_dn = "*"
      org_role = "Viewer"
```

To enable nested groups, add the following code under the `[[servers]]` section :
```yaml
      group_search_filter = "((member:1.2.840.113556.1.4.1941:=%s))"
      group_search_base_dns = ["OU=grafana,OU=Shared-services,DC=devibm,DC=local"]
      group_search_filter_user_attribute = "distinguishedName"
```

### Plugins

To install plugins, you can add the following code under the `grafana` section :

```yaml
## Pass the plugins you want installed as a list.
##
plugins:
  - grafana-piechart-panel
  # - digrich-bubblechart-panel
  # - grafana-clock-panel
```