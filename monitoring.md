# Monitoring

## HPA with custom metrics

pre-requisites:
* prom with the metric
* prom adapter
* hpa

### 1. make the metric available in Prom

#### expose metric from your container

You will probably need an exporter (native from the container you use or you might need to add a sidecar container fetching the metric and exposing it to Prom). You can debug this step by port-forwarding the exposed port in which case the endpoint should respond metrics in "Prom format".

#### configure prom to scrap the endpoint

Then you will need to configure Prom to scrap the endpoint. It used to be confgured in some ConfigMap for old versions of Prom. Now it is declared with CRD resources `PodMonitor`/`ServiceMonitor`. Be careful with the labels set on those two resources: it should match with the labels configured in Prom (`serviceMonitorSelector.matchLabels` and `podMonitorSelector.matchLabels` for ex for the helm repo kube prom stack).
You can debug this with the Prom UI: 
* `/targets` shows scrapped endpoints
* `/graph` can be used to fetch one particular metric - use this to verify that the metric is inside Prom


### 2. make the metric available for k8s resources

#### configure Prom adapter (i.e. custom.metrics.k8s.io)

In order for your HPA (or for any k8s "native" resource) to know about your custom metric, this one has to be exposed through the `custom.metrics.k8s.io` API. This is done by the Prom adapter, a component dedicated for this purpose and which is not necessarily installed with Prometheus.
This adapter translates prom metric format to k8s metric format and make it available through k8s custom metric api.
We have to configure this translation thanks to prom-adapter `rules` declared inside a prom-adapter comfigMap (use prom-adapter values if you installed it with helm).
If your configuration, your new metric should be available on custom metric api. You can debug it using 2 different commands:
* `kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq` > returning all metrics available with their parameters
* `kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/<path to your metric>" | jq` f ex `kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/su/pods/*/phpfpm_processes_total" | jq` > this is the endpoint used by the HPA. It returns the value of your metric

```yaml
  rules:
    custom:
      - seriesQuery: '{__name__="phpfpm_processes_total"}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
            pod: {resource: "pod"}
        name:
          matches: "phpfpm_processes_total"
          as: "phpfpm_processes_total"
        metricsQuery: 'sum(phpfpm_processes_total{state="active"}) by (namespace, pod)'
```

### 3. create the HPA

The hpa is a namespaced resource which means that it will look for the metric using the same namespace.

### 4. external metrics

Sometimes, scaling of a particular resource must rely on a metric external to the resource e.g. scaling of queue consumers relies on the number of messages in the queue which is exposed by the server pods e.g. rabbitmq-server pod. This is when external metrics API comes into play. 
In this case, the prom adapter rule declares an external metric:
```yaml
  rules:
    external:
      - seriesQuery: 'rabbitmq_detailed_queue_messages_ready{vhost="/",queue="chili_local_print_pdf"}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
        name:
          matches: "^rabbitmq_detailed_queue_messages_ready$"
          as: "rabbitmq_queue_messages_ready_chili_local_print_pdf"
        metricsQuery: 'sum(rate(rabbitmq_detailed_queue_messages_ready{vhost="/",queue="chili_local_print_pdf"}[1m])) by (pod)'
```
And you can debug using:
* `kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq`

Troubleshooting :
* the `seriesQuery` and `metricsQuery` should both return results when copy/pasting it into prometheus
* `name.matches` and `name.as` is a way to change the name of the exposed metric dynamically. We can simply take the metrics resulting from `seriesQuery` and `matches` should match it as a regular expression. `as` is the way you want your metric to be named from prom adapter. Inside `as` you can use arguments you've captured with your regex in `matches`
