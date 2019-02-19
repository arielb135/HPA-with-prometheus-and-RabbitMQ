# HPA-with-prometheus-and-RabbitMQ
Autoscale a deployment based on a custom metric from prometheus (RabbitMQ) tutorial:


**Few things that are important to do: ** 

**Install prometheus-adapter+ metrics server**:
https://github.com/helm/charts/tree/master/stable/metrics-server
https://github.com/helm/charts/tree/master/stable/prometheus-adapter

- Note - in the adapter's values.yaml - you'll find the prometheus URL you need to adjust (in port 9090), as the official prometheus operator creates dynamically some endpoints called (X-operated) - i've used that, example:
```
prometheus:
  url: http://prometheus-operated.monitoring.svc
  port: 9090
```
You can check the logs to see if it's able to connect to prometheus, also note that i've deployed prometheus operator in the **monitoring** namespace (that's why the dns is monitoring.svc), if it's in default - do http://prometheus-operated.default.svc, etc...

1. Make sure prometheus scrapes all namespaces, if you're using prometheus helm chart, add this:
```
    ## Namespaces to be selected for PrometheusRules discovery.
    ## If unspecified, only the same namespace as the Prometheus object is in is used.
    ##
    ruleNamespaceSelector: 
      any: true
```
2. **always always always** - make sure your custom prometheus rules contain the same labels in prometheus' ruleSelector:
```
    ruleSelector:
    ## Example which select all prometheusrules resources
    ## with label "prometheus" with values any of "example-rules" or "example-rules-2"
    # ruleSelector:
    #   matchExpressions:
    #     - key: prometheus
    #       operator: In
    #       values:
    #         - example-rules
    #         - example-rules-2
    #
    ## Example which select all prometheusrules resources with label "role" set to "example-rules"
    # ruleSelector:
      matchLabels:
        role: my-rules
        prometheus: kube-prometheus
```
3. Go to prometheus UI for example, decide which custom query you want to build - and test it:
![image](https://user-images.githubusercontent.com/33662254/53008408-fe91d700-3441-11e9-9420-0352f0d4ed84.png)

I've decided to do average of ready messages in the last 5 minutes.

4. Create a **PrometheusRule** object and define the custom metric:
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: someApp
  namespace: default   
  labels:
    prometheus: kube-prometheus
    role: my-rules
    app: someApp
spec: 
  groups:
  - name: myCustomRule
    rules:    
    - record: messages_waiting_in_queue
      expr: avg(avg_over_time(rabbitmq_queue_messages_ready{queue="myQueue"}[5m]))
      labels:
        namespace: default
        service: newCustomService
```
Few notes:

- The labels must contain the values you did in rule selector in the prometheus (rule + prometheus in my side)
- The spec's labels namespace must be the same namespace where the deployment and the hpa are deployed - default in my case.
- the service name can be anything.

After applying the above, you can test it worked in few ways:

- go to prometheus operator logs - see that no errors occured on the log when it scraped the new rule (typos, or wrong expressions...)
- go to the shell of the prometheus operator, _/etc/prometheus/rules/XXXrulefiles-0/_ - see that you can see the new yaml file there (of the new rule) - and it's not empty.

Now, perform: **kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1**, search for your new rule under services:
```
{
      "name": "services/newCustomService",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
```

Perform and check the new custom metric with:
**kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/newCustomService/messages_waiting_in_queue"**

you can see it here:
```
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/newCustomService/messages_waiting_in_queue"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Service",
        "namespace": "default",
        "name": "newCustomService",
        "apiVersion": "/v1"
      },
      "metricName": "messages_waiting_in_queue",
      "timestamp": "2019-02-19T10:48:37Z",
      "value": "2"
    }
  ]
}
```

* Note that we used the defined service name newCustomService (it doesn't matter what name is it) - and we will use it in the HPA.

5. **Finally create the HPA**

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:  
  labels:
    app: someApp
  name: someApp
  namespace: default    
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: someApp
  metrics:
    - type: Object
      object:
        target:
          apiVersion: v1
          kind: Service
          name: newCustomService
        metricName: messages_waiting_in_queue
        targetValue: 100
```

Few important notes:
- It must be in the same namespace of the PrometheusRule
- about the metrics, the **apiVersion** is what you received in the "apiVersion": "/v1" when you queried the metric
- the kind must be "Service"
- the name of the service is what you defined before (unrelated to "Kubernetes service")

In the above, the autoscaler will kick in if there are on average more than 100 messages ready in queue for the last 5M.

Enjoy
