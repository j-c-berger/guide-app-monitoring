# Application Monitoring on Openshift with Prometheus and Grafana

## Introduction
When it comes to application monitoring on Openshift, Openshift Prometheus Cluster Monitoring feature is usually the first showing up on the google search result. It is an Openshift feature that can be installed by one command via Openshift ansible playbooks just like openshift-logging. It packages both Prometheus Operator and Grafana in a single installation. It sounds like an one-click approach for all your monitoring needs, except well, it is not! The Prometheus Cluster Monitoring is not meant to be modified in anyways by the user when it comes to the scrape targets and other Prometheus configuration. Its purpose is to monitor a selective of Openshift and Kubernetes namespaces and provide a dashboard view on Grafana to show users how their Openshift or Kubernetes system is doing. As a result, you cannot rely on this feature for the day-to-day application monitoring needs. For application monitoring on Kubernetes, there are currently two approaches to setup Prometheus on your cluster: 1. Prometheus Operator with ServiceMonitor and 2. Legacy Prometheus deployment with annotations. 


## Deploy A Sample Application with MP Metrics Endpoint
Before diving into Prometheus deployment instructions, make sure there is a running application that has a service endpoint for outputing metrics in Prometheus format. 

In this guide, let's assume such app has been deployed to the Openshift cluster inside a project/namespace called **myapp** and the prometheus metrics endpoint is exposed on path **metrics** with the application port of **9080**.

## Deploy Prometheus - Prometheus Operator

The Prometheus Operator is an open source project from a company called CoreOS, which was later acquired by Red Hat. It's starting to become the de facto standard for Prometheus deployments on Kubernetes system. When Prometheus Operator is installed on the Kubernetes system, user no longer needs to deal with prometheus configuration by hands. Instead, they will ServiceMonitors objects each of the service endpoint that needs to be monitored, which makes maintaining the Prometheus stack a lot easier in daily operation. An overview architecture of the Prometheus is shown below:

![Prometheus Operator](https://miro.medium.com/max/1400/1*R7cnpxuu-vWYkq7ciAPA_w.png "Prometheus Operator Architecture")

There are two ways to install Prometheus Operator. One is through Openshift Operator Lifecycle Manager, which is still in its technology preview phase in release 3.11. This approach will install an older version of Prometheus Operator that's supported by Openshift and Redhat. Another approach is to install Prometheus Operator by following the guide from the project's git repository at [here](https://github.com/coreos/prometheus-operator). Since OLM is still at its tech preview stage and requires a Red Hat subscription account for installation, this guide will show the installation without OLM to target a larger audience base. 

### Prometheus Operator Installation without OLM

The following guide largely based on the [Getting Started](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md) guide maintained by CoreOS team, with the inclusion of commands needed for each steps and minor tweaks for an Openshift deployment.   

Let's start by cloning the Prometheus Operator project repository.

Clone the repository
```
[root@rhel7-openshift ~]# git clone https://github.com/coreos/prometheus-operator
```

Open the bundle.yaml file and change all instances **namespace: default** to the namespace where you want to deploy the prometheus operator. In this example, we'll use **namespace: prometheus-operator**.

Save the file and deploy the prometheus-operator using the following command.
```
[root@rhel7-openshift ~]# oc apply -f bundle.yaml
```

You might encounter an error message like the one below when running the command.
```
Error creating: pods "prometheus-operator-5b8bfd696-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 65534: must be in the ranges: [1000070000, 1000079999]]
```

Simply change **runAsUser: 65534** field to a value that is in the range from the error message. In this case, let's just set **runAsUser: 1000070000**. Save the bundle file and redeploy promtheus-operator.
```
[root@rhel7-openshift ~]# oc delete -f bundle.yaml
[root@rhel7-openshift ~]# oc apply -f bundle.yaml
```

Create a **service_monitor.yaml** file that defines a ServiceMounitor resource. A ServiceMonitor defines a service end point that should be monitored by the Prometheus instance. In this example, the application with label **app: myapp** from namespace **myapp**, and metrics endpoints defined in **spec.endpoints** is to be monitored.
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: myapp-monitor
  name: myapp-monitor
  namespace: myapp
spec:
  endpoints:
    - interval: 30s
      path: /metrics
      port: 9080-tcp
  namespaceSelector:
    matchNames:
      - myapp
  selector:
    matchLabels:
      app: myapp

```

Apply the servicemonitor.yaml file to create the ServiceMonitor resource.
```
[root@rhel7-openshift ~]# oc apply -f service_monitor.yaml
```

Next, define a Prometheus resource that will scrape the targets defined in the ServiceMonitor resource. 
Create a prometheus.yaml file that aggregates all the files from git repository directory prometheus-operator/example/rbac/prometheus/. Make sure to change the **namespace: default** to **namespace: prometheus-operator** .
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: prometheus-operator
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus-operator
spec:
  serviceAccountName: prometheus
  serviceMonitorNamespaceSelector:
    matchLabels:
      prometheus: monitoring
  serviceMonitorSelector:
    matchExpressions:
      - key: k8s-app
        operator: Exists
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: prometheus-operator
spec:
  type: NodePort
  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    prometheus: prometheus
```

Apply the prometheus yaml file to deploy the Prometheus service. After all the resources are created, apply the Prometheus Operator bundle yaml file again.

```
[root@rhel7-openshift ~]# oc apply -f service_monitor.yaml
[root@rhel7-openshift ~]# oc apply -f bundle.yaml
```

Verify that the Prometheus services have successfully started. The prometheus-operated service is created automatically by the prometheus-operator, and is used for registering all deployed prometheus instances. 

```
[root@rhel-2EFK ~]# oc get svc -n prometheus-operator
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
prometheus            NodePort    172.30.112.199   <none>        9090:30342/TCP   19h
prometheus-operated   ClusterIP   None             <none>        9090/TCP         19h
prometheus-operator   ClusterIP   None             <none>        8080/TCP         21h
```

Expose the prometheus-operated service so that we can access the Prometheus console externally.
```
[root@rhel-2EFK]# oc expose svc/prometheus-operated -n prometheus-operator
[root@rhel-2EFK]# oc get route -n prometheus-operator
NAME         HOST/PORT                                                 PATH      SERVICES     PORT      TERMINATION   WILDCARD
prometheus   prometheus-prometheus-operator.apps.9.37.135.153.nip.io             prometheus   web                     None
```

Visit the **prometheus** route and go to the Prometheus **targets** page. At this point, the page should be empty with no endpoints being discovered. There is no need to worry as that's expected at this point. There is one more step to get the targets showing up. Going back to visit prometheus.yaml file again and pay attention to both **serviceMonitorNamespaceSelector** and **serviceMonitorSelector** fields in the yaml file. The ServiceMonitor needs to satisfy the matching requirement for both selectors before it can be picked up by the Prometheus service. In this case, our service monitor has the *k8s-app* label, but the target namespace "myapp" is missing the required *prometheus: monitoring* label.
```
  serviceMonitorNamespaceSelector:
    matchLabels:
      prometheus: monitoring
  serviceMonitorSelector:
    matchExpressions:
      - key: k8s-app
        operator: Exists
```
Add the label to "myapp" namespace.
```
[root@rhel-2EFK]# oc label namespace myapp prometheus=monitoring
```

Now you should see the Prometheus targets page is picking the target endpoints. If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make prometheus-operator project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus-operator
```



## Legacy Prometheus deployments

For users who just migrated their applications to Openshift and are used to handcrafting their own Prometheus configuration file, Prometheus Operator is not the only option for Prometheus deployments. it's still possible to deploy Prometheus the old fashioned way without too much headeache thanks to the eaxmple yaml file provided by Openshift
```
[root@rhel7-openshift ~]# oc new-project Prometheus
```

First of call, deploy the Prometheus using the sample prometheus yml file from [here](https://github.com/openshift/origin/tree/master/examples/prometheus)
```
[root@rhel7-openshift ~]# oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/prometheus/prometheus.yaml -p NAMESPACE=prometheus
```

Edit the "prometheus" ConfigMap resource from prometheus namespace. 
```
[root@rhel7-openshift ~]#  oc edit configmap/prometheus -n prometheus
```

Remove all exiting jobs and add the following job. 
```
...
scrape_configs:
...
      # Scrape config for API servers.
      #
      # Kubernetes exposes API servers as endpoints to the default/kubernetes
      # service so this uses `endpoints` role and uses relabelling to only keep
      # the endpoints associated with the default/kubernetes service using the
      # default named port `https`. This works for single API server deployments as
      # well as HA API server deployments.
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
...
```

Kill the exsiting prometheus pod or better yet reload the prometheus service gracefully using the command below for the new new configuration to take effect.
```
[root@rhel7-openshift ~]# oc exec prometheus-0 -c prometheus -- curl -X POST http://localhost:9090/-/reload
```

Make sure the monitored application's pods are started with the following annotations as specified in the prometheus ConfigMap's scrape_configs. 
```
    prometheus.io/path: /metrics
    prometheus.io/port: '9080'
    prometheus.io/scrape: 'true'
```

Verify the scrape target is up and available in Prometheus by visitng Promethues **Console -> Status -> Targets**.

If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make prometheus project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus
```

## Deploy Grafana

Regardless which approach was used for Prometheus deployment on Openshift, the end game is always to use the more feature riched Grafana for dashboard and visualization of the metrics. The installation of Grafana is mostly straight forward using the sample grafana yaml file provided by Openshift origin git repository. There is however some steps involved to add prometheus endpoint reachable as a datasource on Grafana.

First thing first, create a new project called grafana.
```
[root@rhel-2EFK ~]# oc new-project grafana
```

Deploy Grafana using the grafana.yaml from openshift origin repository.
```
[root@rhel-2EFK ~]# oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/grafana/grafana.yaml -p NAMESPACE=grafana
```

Grant grafana service account view access to the prometheus (or prometheus operator) namespace 
```
[root@rhel-2EFK ~]# oc policy add-role-to-user view system:serviceaccount:grafana:grafana -n prometheus
```

In order for Grafana to add existing prometheus datasource in Openshift, one would need to define the datasource in a ConfigMap resource under grafana namespace. Create a ConfigMap called 'grafana-datasources'. For the key-value pair of the ConfigMap, enter '**datasources.yaml**' for key, and enter the following as the value.
```
apiVersion: 1
datasources:
  - name: "OCP Prometheus"
    type: prometheus
    access: proxy
    url: https://route.to.prometheues.app
    basicAuth: false
    withCredentials: false
    isDefault: true
    jsonData:
        tlsSkipVerify: true
        "httpHeaderName1": "Authorization"
    secureJsonData:
        "httpHeaderValue1": "Bearer [grafana-ocp token]"
    editable: true
```

The \[grafana-ocp token\] can be acquired by the following command
```
[root@rhel-2EFK ~]# oc sa get-token grafana
```
  
Add the config map the application grafana-ocp and mount to '/usr/share/grafana/datasources'

Save and test the data source. You should see 'Datasource is working'. Good job, it is now possible to consume all the application metrics gathered by Prometheus on Grafana dashboard.
  
