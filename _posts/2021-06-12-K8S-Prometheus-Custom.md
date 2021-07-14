# Kubernetes HPA : ExternalMetrics+Prometheus

[![Emir Özbir](https://miro.medium.com/fit/c/96/96/1*CKMOHyefQljjicivFJs7lA.jpeg)](https://emirozbirdeveloper.medium.com/?source=post_page-----acb1d8a4ed50--------------------------------)

[Emir Özbir](https://emirozbirdeveloper.medium.com/?source=post_page-----acb1d8a4ed50--------------------------------)[Follow](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2F1985aeefa8b6&operation=register&redirect=https%3A%2F%2Fblog.kloia.com%2Fkubernetes-hpa-externalmetrics-prometheus-acb1d8a4ed50&user=Emir Özbir&userId=1985aeefa8b6&source=post_page-1985aeefa8b6----acb1d8a4ed50---------------------follow_byline-----------)

[Dec 11, 2019](https://blog.kloia.com/kubernetes-hpa-externalmetrics-prometheus-acb1d8a4ed50?source=post_page-----acb1d8a4ed50--------------------------------) · 7 min read



# Hello everyone,

Kubernetes has an extendable architecture on itself. This architecture is  including auto-scaling and related to some requirements we need to scale our application according to the external resources metrics rather than default cpu and ram usage.

For example, when my dB connection increased I need to scale my cache deployment replicas and today, we will learn **how the Kubernetes autoscaler metrics works and how we can handle this issue E2E** (a little Prometheus )?

# Let’s Start !

I divided the article into 3 main headings;

- First of all, I will talk about **HPA** and **METRIC** server relationships in a basic way.
- In the second chapter, we will integrate **PROMETHEUS** and **prometheus-adapter** with the **k8s**. We will then discuss two types of **metrics;** **custom** and **external.**
- In the third chapter, we will look at the question of how we can implement our applications via HPA via external metrics.

# HPA and METRIC SERVER

- 1 kubernetes cluster (1 master 1 node is sufficient [preferably spot]): D
- 1 metric server
- 1 deployment object and 1 hpa implementation

# Kubernetes Metric Server

MetricServer Kubernetes is a structure that collects metrics from objects such as  pods, nodes according to the state of CPU, RAM and keeps them in time.

In the Kubernetes world, we have heard the word addons a lot, and you can  think of packages that can extend the existing kubernetes-api.

Metric-Server can be installed in the system as an addon. Under Stable repo, you can  take and install it directly from the repo using the helm chart or the  command below.

git clone[ https://github.com/kubernetes-incubator/metrics-server](https://github.com/kubernetes-incubator/metrics-server)cd metric-server && kubectl apply -f deploy/1.8+/

When the Metric server is installed directly, it is installed under the cube-system namespace.

Is it enough to just collect the metrics listens to them and according to  him in the action I have mentioned above HPA is coming out.

We completed the first part in the article under the first phase, now  let’s create a simple deployment object and the HPA structure attached  to it and observe its behavior.

\## deployment.yamlapiVersion: extensions/v1beta1

```
kind: Deployment
metadata:
  name: deployment-firstspec:
  replicas: 2
  template:
    metadata:
      labels:
        app: deployment-first
  spec:
    containers:
      - name: deployment-first
        image: nginx
        imagePullPolicy: Always
      ports:
        - containerPort: 80
      protocol: TCP
      resources:
        requests:
         cpu: "1m"
        limits:
         cpu: "100m
```

In the example above we are doing a basic deployment on an nginx then we  will see its implementation on hpa. Then we will connect to the  corresponding deployment object on the hpa.

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-cpu-hpaspec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: deployment-first
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
  resource:
    name: cpu
    targetAverageUtilization: 80
```

Let’s run these commands :

```
kubectl apply -f deployment.yaml && kubectl apply -f hpa.yaml
```

And get the output of the results :

```
NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGEnginx-cpu-hpa Deployment/deployment-first 100%/80% 2 10 7 25s
```

When we look at the above outputs, we have been able to collect metrics on  the basis of CPU and even our application has been scale.

![img](https://miro.medium.com/max/1000/0*O2hbBahk7Hqxp0kt)

In the diagram above, first the resource is created via metrics.k8s.io on the cubicle and pushed to metric api.

When viewed, it receives data from the metric api from HPA.

Consequently, it increases or decreases the number of pod replica from cube-apiserver from HPA. By default, Metric api can collect metrics such as CPU / RAM, but how can we implement our applications on different metrics other  than the above.

Now let’s move on to the second title of our article.

# Prometheus and Custom Metrics

Prometheus is a metric database that holds the metrics sent to it in the form of time series for us.


![img](https://miro.medium.com/max/1000/0*Wv7Bo7v0gCG0AK6q)

If we look at the diagram above, prometheus can pull the data from the which application share the metrics .

Metrics that can be collected from Kubernetes can also be metrics by exporters  that also include metrics in your own application in a custom way.

Now let’s first set the prometheus to the cluster we have. I’m getting helm’s stable charts repo to do it myself.

git clone[ https://github.com/helm/charts.git](https://github.com/helm/charts.git)

cd charts/stable

helm install — name prometheus ./prometheus

I keep the Prometheus bet on the prometheus adapter. Two important issues appear when examining the prometheus adapter file under a stable repo.

![img](https://miro.medium.com/max/1000/0*5NP1qv7gQLXOztag)

If we go back to Kubernetes, CPU RAM node connected components can collect metrics in itself, but when we look at it, we need the get metrics  related to our applications like http_connections, error_rate of  requests, session_count and etc .

This is where the Prometheus adapter comes into play and collects the  metrics gathered in prometheus through the kubernetes metric apis under  custom and external metrics.

# Custom/External Metric API

Custom metrics are shared by our exporters as a metrics on kubernetes **custom_metric_api**.

```
helm install — name prometheus-adapter ./prometheus-adapter
```

We also get the external metrics, which is the main reason for the  problem, through this adapter. If we take a look at the Prometheus  adapter.

Stackdriver is generally recommended to open external metrics, but I will use the **DirectXMan12 / k8s-prometheus-adapter** project.

After the Prometheus adapter is installed, each metric counterpart that comes to prometheus is reflected in the custom-metric-api-server.

```
kubectl get — raw “/apis/custom.metrics.k8s.io/v1beta1/” | jq
```

I have set up a mongo-exporter for testing purposes and in addition,  metric from the mongo-exporter via prometheus adapter reaches  custom.metrics.api after prometheus.

Below are the outputs of mongo related metric custom api.

```
{ 
   "name":"namespaces/mongodb_mongod_metrics_repl_network_ops",
   "singularName":"",
   "namespaced":false,
   "kind":"MetricValueList",
   "verbs":[ 
      "get"
   ]
}
```

Custom Metric Api is now integrated into the kubernetes API with **apiregistration.k8s.io/v1beta1** endpoint to Kubernetes.

Here we can now connect HPA to the custom metric api and connect it to our own metric.

Let’s check the example at the below , and now we can fetch the redis-connections metric from prometheus export of redis .

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscale:
metadata:
  name: redis-hpa
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: redis
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: redis-connections
      targetAverageValue: 10
```

However, the main problem here is that custom metrics cannot affect any other  pods, since their metrics are only collected for redis pods.

# Problem :

Applications have different metrics within themselves. These can be CPU, memory, I / O. But the **external** metrics can affect our application. These can be redis_connection,  mongodb_connection or count of the member on the shop . These metrics  has a common point , metrics are not coming from inside of application,  these metrics are coming from external components related to the  application .

For example ;

![img](https://miro.medium.com/max/1000/0*8VnoCo5zURLMWIDH)

When the connection count on Mongodb increases, there may be three  situations in which my application on the web-api should self-implement. In this context, we can only use the mongo-exporter metric for mongo  pods when we clarify this by HPA.

For this kind of needs, we collect them under **external.metrics.k8s.io** in kubernetes.

What we mean is that the metrics that only concern a pod can be shared. So we can use these metrics in other components.

To do this, we need to configure a prometheus-adapter.

```
{{- if .Values.rules.external }}apiVersion: apiregistration.k8s.io/v1beta1kind: APIServicemetadata:labels:app: {{ template “k8s-prometheus-adapter.name” . }}name: v1beta1.external.metrics.k8s.io
```

I am adding to the values.yaml side of a sample rule that I need to add some rules to work on external api as seen above.

```
- seriesQuery: ‘{__name__=~”mongodb_connections”}’resources:overrides:kubernetes_namespace: {resource: “namespace”}kubernetes_pod_name: {resource: “pod”}name:matches: “”as: “mongodb_current_connection”metricsQuery: sum( mongodb_connections{state=”current”,} )
```

Prometheus by each metrigi gelicek no longer specified in the query now I share externally.

Normally the prometheus shares the number of available connections through its  mongo_exporter structure, which can be around 8000.

I have questioned the current number of connections by adding metric Query to prometheus adapter external rule as above.

I can now control the external api.

```
kubectl get — raw \ 
"/apis/external.metrics.k8s.io/v1beta1/namespaces/monitoring/mongodb_current_connection" | jq
```

## That result of external metric value :

```
{ 
   "kind":"ExternalMetricValueList",
   "apiVersion":"external.metrics.k8s.io/v1beta1",
   "metadata":{ 
      "selfLink":"/apis/external.metrics.k8s.io/v1beta1/namespaces/monitoring/mongodb_current_connection"
   },
   "items":[ 
      { 
         "metricName":"mongodb_current_connection",
         "metricLabels":{ 

         },
         "timestamp":"2019–10–05T22:25:13Z",
         "value":"9"
      }
   ]
}
```

I can now implement my application based on the number of connections in  mongodb. Hpa As mentioned above, Custom metrics can now be used as  external metrics in the entire cluster universe.

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: app-server-mongo-conn-hpaspec:
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: External
  
  external:
    metricName: mongodb_current_connection
    targetValue: 30scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: deployment-first
```

You can check the final status both by the kubernetes and to make sure mongo_connections are calculated correctly

```
kubectl describe hpa app-server-mongo-conn-hpa
```

**and connect to the mongodb**

```
var status = db.serverStatus()status.connections{“current” : 9, “available” : AAAAA}
```

# Conclusion

Now in the above universe, I have become measurable with a metric that my own application has exported.

The biggest problem that prevents us here is the fact that custom metrics  only work with the relevant pod source, because the independent  components that work together always affect them in the world of  microservice.