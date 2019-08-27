# Application Monitoring on Openshift with Prometheus and Grafana

## Introduction
When it comes to application monitoring on Openshift, the Prometheus Cluster Monitoring offered by Openshift is usually the first thing showing up on the Google search result. It is an Openshift feature that can be installed by one command via Openshift Ansible playbooks just like openshift-logging. It packages both Prometheus Operator and Grafana in a single installation. It sounds like a one-click approach for all your application monitoring needs, except well, it is not! The Prometheus Cluster Monitoring is designed to monitor a selective few of predefined Openshift and Kubernetes namespaces. Red Hat openly stated that modifying the Prometheus scrape targets to include namespaces outside of predefined subsets is not supported. As a result, you cannot rely on this feature for the day-to-day application monitoring needs. For application monitoring on Openshift, the user should set up their own Prometheus and Grafana deployments. This guide explores two approaches in setting up Prometheus on Openshift. The first approach is via Prometheus Operator and Service Monitor, which is arguably the most popular and future proof way of setting up Prometheus on a Kubernetes system. The second approach is the legacy way of deploying Prometheus on Openshift without the Prometheus Operator. 


## Deploy A Sample Application with MP Metrics Endpoint
Before diving into Prometheus deployment instructions, make sure there is a running application that has a service endpoint for outputting metrics in Prometheus format. 

In this guide, it is assumed such app has been deployed to the Openshift cluster inside a project/namespace called **myapp** and the Prometheus metrics endpoint is exposed on path **metrics** with the application port of **9080**.

## Deploy Prometheus - Prometheus Operator

The Prometheus Operator is an open-source project from CoreOS as part of their Kubernetes Operator offering. It is starting to become the standard for Prometheus deployments on Kubernetes system. When Prometheus Operator is installed on the Kubernetes system, users no longer need to deal with Prometheus configuration by hands. Instead, they need to create ServiceMonitor resources for each of the service endpoint that needs to be monitored, which makes maintaining the Prometheus server a lot easier in daily operation. An architecture overview of the Prometheus Operator is shown below:

![Prometheus Operator](https://coreos.com/sites/default/files/inline-images/p1.png "Prometheus Operator Architecture")

There are two ways to install Prometheus Operator. One is through Openshift Operator Lifecycle Manager or OLM, which is still in its technology preview phase in release 3.11 of Openshift. This approach will install an older version of Prometheus Operator that is supported by Red Hat. Another approach is to install Prometheus Operator by following the guide from the Prometheus Operator's git repository at [here](https://github.com/coreos/prometheus-operator). Since OLM is still at its tech preview stage and requires a Red Hat subscription account for installation, this guide will show the installation without OLM to target a larger audience base. The guide will be updated with OLM approach when Kabanero officially adopts Openshift 4.x a few months from now.

### Prometheus Operator Installation without OLM

The following guide is largely based on the [Getting Started](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md) guide maintained by the CoreOS team, with the inclusion of Openshift commands needed to complete each step.   

Let's start by cloning the Prometheus Operator project repository.

Clone the repository
```
[root@rhel7-openshift ~]# git clone https://github.com/coreos/prometheus-operator
```

Open the bundle.yaml file and change all instances of **namespace: default** to the namespace where you want to deploy the prometheus operator. In this example, let's use **namespace: prometheus-operator**. You might need to create a new namespace called **prometheus-operator** if not present on your Openshift cluster.

Save the file and deploy the prometheus-operator using the following command.
```
[root@rhel7-openshift ~]# oc apply -f bundle.yaml
```

You might encounter an error message like the one below when running the command.
```
Error creating: pods "prometheus-operator-5b8bfd696-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 65534: must be in the ranges: [1000070000, 1000079999]]
```

Simply change **runAsUser: 65534** field to a value that is in the range specified in the error message. In this case, let's just set **runAsUser: 1000070000**. Save the bundle file and redeploy promtheus-operator.
```
[root@rhel7-openshift ~]# oc delete -f bundle.yaml
[root@rhel7-openshift ~]# oc apply -f bundle.yaml
```

Create a file called **service_monitor.yaml**  that defines a ServiceMounitor resource. A ServiceMonitor defines a service endpoint that needs to be monitored by the Prometheus instance. In this example, the application with label **app: myapp** from namespace **myapp**, and metrics endpoints defined in **spec.endpoints** will be monitored by Promtheus.
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

Next, define a Prometheus resource that can scrape the targets defined in the ServiceMonitor resource. 
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

Apply the Prometheus yaml file to deploy the Prometheus service. After all the resources are created, apply the Prometheus Operator bundle yaml file again.

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

Visit the **prometheus** route and go to the Prometheus **targets** page. At this point, the page should be empty with no endpoints being discovered. There is no need to worry as that's expected. There is one more step to get the targets showing up. Going back to the prometheus.yaml file again and pay attention to both **serviceMonitorNamespaceSelector** and **serviceMonitorSelector** fields in the yaml file. The ServiceMonitor needs to satisfy the matching requirement for both selectors before it can be picked up by the Prometheus service. In this case, our ServiceMonitor has the **k8s-app** label, but the target namespace "myapp" is missing the required **prometheus: monitoring** label.
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

For users who just migrated their applications to Openshift and are used to handcrafting their own Prometheus configuration file, Prometheus Operator is not the only option for Prometheus deployments. it is still possible to deploy Prometheus the old fashioned way without too much headache thanks to the eaxmple yaml file provided by Openshift.
```
[root@rhel7-openshift ~]# oc new-project Prometheus
```

First of all, deploy the Prometheus using the sample prometheus yml file from [here](https://github.com/openshift/origin/tree/master/examples/prometheus)
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

Kill the existing Prometheus pod or better yet reload the Prometheus service gracefully using the command below for the new configuration to take effect.
```
[root@rhel7-openshift ~]# oc exec prometheus-0 -c prometheus -- curl -X POST http://localhost:9090/-/reload
```

Make sure the monitored application's pods are started with the following annotations as specified in the prometheus ConfigMap's scrape_configs. 
```
    prometheus.io/path: /metrics
    prometheus.io/port: '9080'
    prometheus.io/scrape: 'true'
```

Verify the scrape target is up and available in Prometheus by visiting Prometheus's web console **Console -> Status -> Targets**.

If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make prometheus project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus
```

## Deploy Grafana

Regardless of which approach was used for Prometheus deployment on Openshift, the end game is always to use the more feature riched Grafana for dashboard and visualization of the metrics. The installation of Grafana is mostly straight forward using the sample grafana yaml file provided by Openshift origin git repository. There is, however, some steps involved to add Prometheus endpoint reachable as a data source in Grafana.

First thing first, create a new project called grafana.
```
[root@rhel-2EFK ~]# oc new-project grafana
```

Deploy Grafana using the grafana.yaml from openshift origin repository.
```
[root@rhel-2EFK ~]# oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/grafana/grafana.yaml -p NAMESPACE=grafana
```

Grant grafana service account view access to the *prometheus* (or *prometheus operator*) namespace 
```
[root@rhel-2EFK ~]# oc policy add-role-to-user view system:serviceaccount:grafana:grafana -n prometheus
```

In order for Grafana to add existing prometheus datasource in Openshift, one would need to define the datasource in a ConfigMap resource under grafana namespace. Create a ConfigMap yaml file called grafana-datasources.yaml. 
```
apiVersion: v1
data:
  datasources.yaml: |-
    apiVersion: 1
    datasources:
      - name: "OCP Prometheus"
        type: prometheus
        access: proxy
        url: http://prometheus-operated-monitoring.apps.9.37.135.153.nip.io
        basicAuth: false
        withCredentials: false
        isDefault: true
        jsonData:
            tlsSkipVerify: true
            "httpHeaderName1": "Authorization"
        secureJsonData:
            "httpHeaderValue1": "Bearer \[grafana-ocp token\]"
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: grafana
```

Apply the yaml file to create the ConfigMap resource.
```
[root@rhel-2EFK ~]# oc apply -f grafana-datasources.yaml
```

The **\[grafana-ocp token\]** can be acquired by the following command.
```
[root@rhel-2EFK ~]# oc sa get-token grafana
```
  
Add the config map to the application grafana and mount to **'/usr/share/grafana/datasources'**.
![AddToApplication](https://github.com/fwji/images/blob/master/configMap.png?raw=true "AddToApplication")


Save and test the data source. You should see 'Datasource is working'. 
![AddToApplication](https://github.com/fwji/images/blob/master/grafana.png?raw=true "AddToApplication")


Good job, it is now possible to consume all the application metrics gathered by Prometheus on Grafana dashboard.
  
  
