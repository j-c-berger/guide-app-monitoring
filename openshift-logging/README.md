# Application Logging on Openshift

Openshift provides a preconfigured EFK(Elasticsearch, FluentD, and Kibana) stack for DevOps to aggregate all container logs. The installation of this EFK stack can be done using the ansible-playbooks included in the [openshift-ansible](https://github.com/openshift/openshift-ansible/tree/release-3.11) repository. Once the installation is completed, the EFK deployments can be found inside the *openshift-logging* namespace of the Openshift cluster.

Our recommended setup for EFK is to have two separated EFK setup. An ops EFK deployments dedicated just for Openshift and Kubernetes logs, and another EFK stack just for user applications. There are several advantages of having an ops EFK stack. For one, it'll be much easier to find applications logs in Kibana, as it won't be polluted with all the kubernetes logs happening all the time. It also provides more flexibility for memory allocations since users can assign system memories for each of the EFK deployments independently. 

# Step 1: Install Openshift and Openshift-logging


## Installing openshift-logging 

The installation of EFK on Openshift is fairly straight forward, as all it requires is an Openshift host [inventory file](https://docs.openshift.com/container-platform/3.11/install/configuring_inventory_file.html) and the openshift-ansible playbook for logging installation. After creating the inventory file, add the following ansible variables to the **[OSEv3:vars]** section of the inventory file. 
```
openshift_logging_use_ops=True
openshift_logging_es_ops_nodeselector={“node-role.kubernetes.io/infra”:“true”}
openshift_logging_es_nodeselector={"node-role.kubernetes.io/infra":"true"}
openshift_logging_es_ops_memory_limit=5G
openshift_logging_es_memory_limit=3G
```

For the sake of simplicity, this guide will just set these five ansible variables. There are many other ansible variables provided by Openshift for advanced fine-tuning the EFK stack. The detailed information about the installation and configuration can be found on Openshift documentation [Aggregating Container Logs](https://docs.openshift.com/container-platform/3.11/install_config/aggregate_logging.html). 

Let's examine each of the variables in detail. Setting **openshift_logging_use_ops=True** instructs the ansible to install two EFKs with a dedicated ops deployment. **openshift_logging_es_nodeselector** and **openshift_logging_es_ops_nodeselector** are two required variables by the openshift-logging ansible-playbook in orer to install Elasticsearch, and people usually just set them to the infra nodes. Lastly, **openshift_logging_es_memory_limit** and **openshift_logging_es_ops_memory_limit** are self-explanatory and can be set according to one's own need. For stable operation, a user should allocate at least 2GB of memory for each of the Elasticsearch deployments. If not specified explicitly, Openshift would allocate 16GB of memory for each of the Elasticsearch deployment by default. It is highly recommended to install openshift-logging on systems with at least 32GB of RAM when **openshift_logging_use_ops** is set to true.

After all the variables are set in the inventory file, the user would just need to execute the ansible-playbook command to install the EFK stack onto its current Openshift Cluster.

```
[root@rhel-2EFK ~]# ansible-playbook -i <inventory_file> openshift-ansible/playbooks/openshift-logging/config.yml -e openshift_logging_install_logging=true
```

The installation process might take a few minutes to complete. Once the installation is completed without any error, you should be able to see the following pods running in the *openshift-logging* namespace.

```
[root@rhel-2EFK ~]# oc get pods -n openshift-logging
NAME                                          READY     STATUS      RESTARTS   AGE
logging-curator-1565163000-9fvpf              0/1       Completed   0          20h
logging-curator-ops-1565163000-5l5tx          0/1       Completed   0          20h
logging-es-data-master-iay9qoim-4-cbtjg       2/2       Running     0          3d
logging-es-ops-data-master-hsmsi5l8-3-vlrgs   2/2       Running     0          3d
logging-fluentd-vssj2                         1/1       Running     1          3d
logging-kibana-2-tplkv                        2/2       Running     6          4d
logging-kibana-ops-1-bgl8k                    2/2       Running     2          3d
```

You should see that elasticsearch, kibana, and curator all have two types of pods: the main one and a secondary set of pods with *-ops* postfix, except for fluentd. This is expected as the split of application logs, and the Openshift operations logs are happening inside that single fluentd instance. All the node system logs and the logs from projects **default**, **openshift**, and **openshift-infra** are considered as operation (or ops) logs and are aggregated to the ops Elasticsearch server. The logs from any other namespaces are aggregated to the main Elasticsearch server.

The openshift-logging ansible-playbook will also expose two routes for external access of the Kibana and ops Kibana web console.

```
[root@rhel-2EFK ~]# oc get routes -n openshift-logging
NAME                 HOST/PORT                             PATH      SERVICES             PORT      TERMINATION          WILDCARD
logging-kibana       kibana.apps.9.37.135.153.nip.io                 logging-kibana       <all>     reencrypt/Redirect   None
logging-kibana-ops   kibana-ops.apps.9.37.135.153.nip.io             logging-kibana-ops   <all>     reencrypt/Redirect   None
```

If you head to the **logging-kibana-ops** url, all the operation logs generated by Openshift and Kubernetes should be visible on Kibana's **Discover** page.   

![Kibana Ops Page](https://github.com/fwji/openshift-logging/blob/master/images/kibana-ops.png?raw=true "Kibana Ops Page")
*Figure 1: Kibana ops page with operation log entries*

# Step 2: View application logs on Kibana

Before using the **logging-kibana** for application logs, make sure the application is already deployed in a namespace that is **NOT** from one of the **default**, **openshift**, and **openshift-infra**. 

In order to fully take advantage of the Kibana's dashboard functionalities, it is recommended to output application in JSON format. Kibana is then able to process the data from each individual field of the JSON object to create some nice customized visualization for that field. 

Visit Kibana dashboard page using the routes URL https://kibana.apps.9.37.135.153.nip.io. Login in using your Openshift user and password, then the page should redirect you to Kibana's **Discover** page where the newest logs of the selected index are being streamed. Select **project.\*** index to view the application logs generated by the deployed application. 

![Kibana Application Page](https://github.com/fwji/openshift-logging/blob/master/images/kibana_app.png?raw=true "Kibana Application Page")
*Figure 2: Kibana page with the application log entries*

The **project.\*** index only contains a set of default fields at the start, which will certainly not include all of the fields from the deployed application's JSON log object. Therefore the index needs to be refreshed to have all the fields from the application's log object available to Kibana.  

To refresh the index, click on the **Management** option on the left pane.

Click on the **Index Pattern**, find **project.\*** in Index Pattern and click on the refresh fields button on the right. Once the Kibana is updated with all the available fields in the **project.\*** index, it's time to import the preconfigured dashboards to view the application logs in some really nice visualization. 

![Refresh Index](https://github.com/fwji/openshift-logging/blob/master/images/refresh_index.png?raw=true)
*Figure 3: Index refresh button on Kibana*

To import dashboard and its associated objects, navigate back to the **Management** page and Click on **Saved Objects**. Click on the **Import** button and select the dashboard file. When prompt, click **Yes, overwrite all** option

Head back to the **Dashboard** page and enjoy the logging on the dashboard imported previously. 
![Dashboard page](https://github.com/fwji/openshift-logging/blob/master/images/dashboard_1.png?raw=true "Dashboard page")
*Figure 4: Kibana dashboard for Open Liberty application logs*

## Reinstalling and uninstalling openshift-logging 

It's always safe to reinstall openshift-logging by re-running the ansible-playbook installation command with updated ansible variables values in the inventory file if changes need to be made for the installed EFK stack. Similarly, if the aggregated containers logging is no longer needed in the current cluster, the same ansible-playbook command can also be used to uninstall the openshift-logging feature by having **openshift_logging_install_logging** set to False.
