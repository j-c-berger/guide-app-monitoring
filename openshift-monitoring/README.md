# openshift-monitoring
This guides used sample OKD templates for deploying bring-your-own Prometheus and Grafana. The links of the original templates repository are listed below:

[https://github.com/openshift/origin/tree/master/examples/prometheus](https://github.com/openshift/origin/tree/master/examples/prometheus)
[https://github.com/openshift/origin/tree/master/examples/grafana](https://github.com/openshift/origin/tree/master/examples/grafana)

## Introduction
When it comes to application monitoring on Openshift, Openshift Prometheus Cluster Monitoring feature is usually the first showing up on the google search result. It is an Openshift feature that can be installed by one command via Openshift ansible playbooks. It packages both Prometheus Operator and Grafana in a single installation. It shoulds like a one-click approach for all your monitoring needs, except well, it is not! The Prometheus Cluster Monitoring is not meant to be modified in anyways by the user when it comes to the scrape targets and other Prometheus configuration. Its purpose is to monitor a selective of Openshift and Kubernetes namespaces and provide a dashboard view on Grafana to show the user how the Kubernetes system is doing. As a result, you cannot rely on this feature for your day-to-day application monitoring needs. For application monitoring on Kubernetes, there are currently two approaches to setup Prometheus on your cluster: 1. standalone Prometheus deployment YAML provide by Openshift and 2. Prometheus Operator. We'll introduce both approaches in this guide.


## Deploy A Sample Application with MP Metrics Endpoint
1. Create a new namespaces 
```
[root@rhel7-openshift ~]# oc new-project ltf
```

2. Create a new application with metric endpoint
```
[root@rhel7-openshift ~]# oc new-app "frankji/liberty-ltf" --name ltf
```

3. Verify metrics endpoint http://approute.nip.io/metrics is up.


## Deploy Prometheus - Standalone deployments

1. Create a new project called Prometheus
```
[root@rhel7-openshift ~]# oc new-project Prometheus
```

2. Deploy Standalone Prometheus with a ServiceAccount that can scrape the entire cluster
```
[root@rhel7-openshift ~]# oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/prometheus/prometheus.yaml -p NAMESPACE=prometheus
```

3. Edit the "prometheus" ConfigMap. Remove all exiting jobs and add the following job. 
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

4. Kill the exsiting prometheus pod or reload the prometheus service gracefully using the command below
```
[root@rhel7-openshift ~]# oc exec prometheus-0 -c prometheus -- curl -X POST http://localhost:9090/-/reload
```

5. Edit the target application deployment yaml file to include the following annotation in the template, and restart the pod.
```
    prometheus.io/path: /metrics
    prometheus.io/port: '9080'
    prometheus.io/scrape: 'true'
```

6. Verify the scrape target is up and available in Prometheus by visitng Promethues Console -> Status -> Targets 

7. If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make Prometheus project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus
```

## Deploy Prometheus - Prometheus Operator

The Prometheus Operator is an opensource project from CoreOS. It's starting to become the de facto standard for Prometheus deployments on Kubernetes system. When Prometheus Operator is installed on the Kubernetes system, usesr no longer needs to deal with prometheus.yml file to define the scrape targets. Instead, ServiceMonitors objects are being created for each of the service endpoint that needs to be monitored, which makes maintaining the Prometheus stack a lot easier in daily operation. An overview architecture of the Prometheus is shown below:

![Prometheus Operator](https://miro.medium.com/max/1400/1*R7cnpxuu-vWYkq7ciAPA_w.png "Prometheus Operator Architecture")

There are two ways to install Prometheus Operator. One is through Openshift Operator Lifecycle Manager, which is still in its technology preview phase in Openshift 3.11. This approach will install an older version of Prometheus Operator that's supported by Openshift and Redhat. Another approach is to install Prometheus Operator using the bundle.yaml file from its official git repository at [Here](https://github.com/coreos/prometheus-operator). 

### Prometheus Operator Installation via OLM

* Make sure that you have a Redhat customer portal user id before proceeding to the following sections

#### Install Prometheus Operator

1. OLM should be installed on your cluster if it has not been setup. Follow [this guide](https://docs.openshift.com/container-platform/3.11/install_config/installing-operator-framework.html) on Openshift to install OLM. 

2. Create a new project called prometheus
```
[root@rhel7-openshift ~]# oc new-project prometheus
```

3. Go to Cluster Console page of the Openshift web portal
![Cluster Console Page](https://github.com/fwji/images/blob/master/cluster-console.png?raw=true "Cluster Console Page")

4. Click on the Operators item and select Catalog Sources link

5. Scroll down to the bottom and click on the *Create Subscription* button of the Prometheus Operator

6. Change the namespace to *prometheus* and click *Create*
```apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  generateName: prometheus-
  namespace: prometheus
spec:
  source: rh-operators
  name: prometheus
  startingCSV: prometheusoperator.0.22.2
  channel: preview
```
#### Deploy a Prometheus Server

1. Click the *Cluster Service Version* on the left panel and click on the Prometheus Operator instance we just created

2. Select Create New Prometheus from the drop-down button. 

![New Prometheus](https://github.com/fwji/images/blob/master/prometheus_server.png?raw=true "New Prometheus")

3. Choose a name for your prometheus server and change the namespace to "promtheus"
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-server
  labels:
    prometheus: k8s
  namespace: prometheus
spec:
  replicas: 2
  version: v2.3.2
  serviceAccountName: prometheus-k8s
  securityContext: {}
  serviceMonitorSelector:
    matchExpressions:
      - key: k8s-app
        operator: Exists
  ruleSelector:
    matchLabels:
      role: prometheus-rulefiles
      prometheus: k8s
  alerting:
    alertmanagers:
      - namespace: prometheus
        name: alertmanager-main
        port: web
```

4. Click on the *Create* button

5. Make sure the Prometheus server pods are up and running
```
[root@rhel-2EFK ~]# oc get pods -n prometheus
NAME                                   READY     STATUS    RESTARTS   AGE
prometheus-operator-7fccbd7c74-5cz24   1/1       Running   0          3d
prometheus-server-0                    3/3       Running   1          3d
prometheus-server-1                    3/3       Running   1          3d
```
6. Expose Prometheus console so that it can be accessed externally.
```
oc expose svc/prometheus-operated -n prometheus
```

#### Create a ServiceMonitor to scrape the metrics endpoint

1. Make sure the target application's metrics endpoint is up and running.

2. Click the *Cluster Service Version* on the left panel and click on the Prometheus Operator instance we just created

3. Select *Create New -> Service Monitor*
![New Service Monitor](https://github.com/fwji/images/blob/master/service_monitor.png?raw=true)

4. Edit the Service Monitor YAML to include target endpoint information. You can use namespaceSelector or selector or both to filter the project or the service endpoint that needs to be scraped by Prometheus server.
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: myapp-monitor
  name: myapp-monitor
  namespace: monitoring
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
5. Click on the *Create* button

6. Lastly, we need to give our Prometheus service view permission all the cluster namespaces
```
oc adm policy add-cluster-role-to-user view system:serviceaccount:monitoring:prometheus-k8s
```

7. Verify the scrape target is up and available in Prometheus by visiting Prometheus Console -> Status -> Targets 

8. If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make Prometheus project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus
```

### Prometheus Operator Installation without OLM

If you don't have a paid Redhat subscription account, or if you want to use the latest release of Prometheus Operator instead of the version offered by openshift, you can install Prometheus Operator without OLM.

Let's start by cloning the Prometheus Operator project repository.

1. Clone the repository
```
[root@rhel7-openshift ~]# git clone https://github.com/coreos/prometheus-operator
```

2. Open the bundle.yaml file and change all instances *namespace: default* to the namespace where you want to deploy the prometheus operator. In this example, we'll use *namespace: prometheus-operator*.

3. Deploy the prometheus-operator. You might have to change *runAsUser: 65534* field to a different value to avoid the user out of range error. 
```
[root@rhel7-openshift ~]# oc apply -f bundle.yaml
```

4. Create a ServiceMonitor.yaml file that defines a ServiceMounitor resource. 
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

5. Apply the ServiceMonitor yaml file
```
[root@rhel7-openshift ~]# oc apply -f service_monitor.yaml
```

6. Lastly, we'll need to define the Prometheus service that will detect the targets defined in the ServiceMonitor resource. 
We'll create a prometheus.yaml file that are mostly based on the sample resources defined under the directory prometheus-operator/example/rbac/prometheus/. Make sure to change the namespace to the one that hosts the Prometheus Operator deployment.
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

7. Apply the prometheus yaml file to deploy the prometheus service. After all the resources are created, apply the Prometheus Operator bundle yaml file again.

```
[root@rhel7-openshift ~]# oc apply -f service_monitor.yaml
[root@rhel7-openshift ~]# oc apply -f bundle.yaml
```

8. Verify the Prometheus services have successfully started. The prometheus-operated service is created automatically by the prometheus-operator once a Prometheus resource is deployed in the same namespace as the Prometheus Operator.

```
[root@rhel-2EFK ~]# oc get svc -n prometheus-operator
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
prometheus            NodePort    172.30.112.199   <none>        9090:30342/TCP   19h
prometheus-operated   ClusterIP   None             <none>        9090/TCP         19h
prometheus-operator   ClusterIP   None             <none>        8080/TCP         21h
```

9. expose the prometheus-operated service so that we can access the Prometheus console externally.
```
[root@rhel-2EFK]# oc expose svc/prometheus-operated -n prometheus-operator
[root@rhel-2EFK]# oc get route -n prometheus-operator
NAME         HOST/PORT                                                 PATH      SERVICES     PORT      TERMINATION   WILDCARD
prometheus   prometheus-prometheus-operator.apps.9.37.135.153.nip.io             prometheus   web                     None
```

10. Visit the Prometheus route and view the targets page. At this point, you should an empty targets page. Don't worry as that's expected, and there is one more step to get the targets showing up. Let's review our Prometheus yaml file again. We defined both serviceMonitorNamespaceSelector and serviceMonitorSelector in the yaml file. That means ServiceMonitor needs to satisfy the matching requirement for both service monitor selectors to be picked up by our Prometheus service. In this case, our service monitor has the *k8s-app* label, but the target namespace "myapp" is missing the required *prometheus: monitoring* label.
```
  serviceMonitorNamespaceSelector:
    matchLabels:
      prometheus: monitoring
  serviceMonitorSelector:
    matchExpressions:
      - key: k8s-app
        operator: Exists
```
We'll have to add the label to "myapp" namespace.
```
oc label namespace myapp prometheus=monitoring
```

11. Now you should see the Prometheus targets page is discovering the target endpoints. If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make Prometheus project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus
```

## Deploy Grafana

1. Create a new project called grafana
```
[root@rhel-2EFK ~]# oc new-project grafana
```

2. Deploy Grafana
```
[root@rhel-2EFK ~]# oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/grafana/grafana.yaml -p NAMESPACE=grafana
```

3. Grant Grafana service account view access to Prometheus
```
oc policy add-role-to-user view system:serviceaccount:grafana:grafana -n prometheus
```

4. In order for Grafana to connect to Prometheus datasource in Openshift, one would need to define the datasource in a ConfigMap under grafana namespace.
  - Create a ConfigMap called 'grafana-datasources'
  - For the key-value pair, enter 'datasources.yaml' for key
  - Enter the following for value
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
   - The \[grafana-ocp token\] can be acquired by the following command
  ```
  oc sa get-token grafana
  ```
  
5. Add the config map the application grafana-ocp and mount to '/usr/share/grafana/datasources'

6. Save and test the data source. You should see 'Datasource is working.'
  
