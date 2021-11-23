# x - PromQL
## Simple request
### Get labels

To get a specific label in grafana variable for example, the pattern is :
label_values(my_metric{type= "xxx", another_label="xxx"},target_label)
example:
```
label_values(kube_namespace_labels{},namespace) 
```
