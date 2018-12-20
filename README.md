# WARNING: This project is no longer maintained. Please consider using the Prometheus compatible kubernetes application: https://github.com/grafana/kubernetes-app

# Grafana App for Kubernetes

[Kubernetes](http://kubernetes.io/) is an open-source system for automating deployment, scaling, and management of containerized applications.

The Grafana Kubernetes App allows you to monitor your Kubernetes cluster's performance. It includes 4 dashboards, Cluster, Node, Pod/Container and Deployment. It also comes with [Intel Snap](http://snap-telemetry.io/) collectors that are deployed to your cluster to collect health metrics. The metrics collected are high-level cluster and node stats as well as lower level pod and container stats. Use the high-level metrics to alert on and the low-level metrics to troubleshoot.

![Container Dashboard](https://raw.githubusercontent.com/raintank/kubernetes-app/master/src/img/cluster-dashboard-screenshot.png)

![Container Dashboard](https://raw.githubusercontent.com/raintank/kubernetes-app/master/src/img/container-dashboard-screenshot.png)

![Node Dashboard](https://raw.githubusercontent.com/raintank/kubernetes-app/master/src/img/node-dashboard-screenshot.png)

### Requirements

1. Currently only has support for **Graphite**. You have to deploy a Graphite server which is accessible from your Kubernetes cluster.
2. For automatic deployment of the collectors, then Kubernetes 1.4 or higher is required.
3. Grafana 4 is required if using TLS Client Auth (rather than Basic Auth).

### Features

- The app uses Kubernetes tags to allow you to filter pod metrics. Kubernetes clusters tend to have a lot of pods and a lot of pod metrics. The Pod/Container dashboard leverages the pod tags so you can easily find the relevant pod or pods.

- Easy installation of collectors, either a one click deploy from Grafana or detailed instructions to deploy them manually them with kubectl (also quite easy!)

- Cluster level metrics that are not available in Heapster, like CPU Capacity vs CPU Usage.

- Pod and Container status metrics. See the [Snap Kubestate Collector](https://github.com/raintank/snap-plugin-collector-kubestate) for more details.

### Cluster Metrics

- Pod Capacity/Usage
- Memory Capacity/Usage
- CPU Capacity/Usage
- Disk Capacity/Usage (measurements from each container's /var/lib/docker)
- Overview of Nodes, Pods and Containers

### Node Metrics

- CPU
- Memory Available
- Load per CPU
- Read IOPS
- Write IOPS
- %Util
- Network Traffic/second
- Network Packets/second
- Network Errors/second

### Pod/Container Metrics

- Memory Usage
- Network Traffic
- TCP Connections
- CPU Usage
- Read IOPS
- Write IOPS
- All Established TCP Conn

### Documentation

#### Installation

1. Use the grafana-cli tool to install kubernetes from the commandline:

```
grafana-cli plugins install raintank-kubernetes-app
```

2. Restart your Grafana server.

3. Log into your Grafana instance. Navigate to the Plugins section, found in the Grafana main menu. Click the Apps tabs in the Plugins section and select the newly installed Kubernetes app. To enable the app, click the Config tab and click on the Enable button.

#### Connecting to your Cluster

1. Go to the Cluster List page via the Kubernetes app menu.

   ![Cluster List in main menu](https://raw.githubusercontent.com/raintank/kubernetes-app/master/src/img/app-menu-screenshot.png)

2. Click the `New Cluster` button.

3. Fill in the Auth details for your cluster.

4. Choose the Graphite datasource that will be used for reading data in the dashboards.

5. Fill in the details for the Carbon host that is used to write to Graphite. This url has to be available from inside the cluster.

6. Click `Deploy`. This will deploy a DaemonSet, to collect health metrics for every node, and a pod that collects cluster metrics.

#### Manual Deployment of Collectors

If you do not want to deploy the collector DaemonSet and pod automatically, then it can be deployed manually with kubectl. If using an older version of Kubernetes than 1.4, you will have to adapt the json files, particularly for the daemonset, and remove some newer features. Please create an issue if you need support for older versions of Kubernetes built in to the app.

The manual deployment instructions and files needed, can be downloaded from the Cluster Config page. At the bottom of the page, there is a help section with instructions and links to all the json files needed.

#### Uninstalling the Collectors (DaemonSet and Pod)

There is an Undeploy button on the Cluster Config page. To manually undeploy them:

```bash
kubectl delete daemonsets -n kube-system snap

kubectl delete deployments -n kube-system snap-kubestate-deployment

kubectl delete configmaps -n kube-system snap-tasks

kubectl delete configmaps -n kube-system snap-tasks-kubestate
```

#### Deploying on Google Cloud Contaner (GKE)

For those who have your Kubernetes cluster deploy automatically on Google Cloud platform via Kubernetes Engine, here is the simple step you can follow to deploy to your cluster, this is for those who wants to deploy everything on a single Kubernetes cluster. For those who want to do otherwise, this basic steps might be helpful.

**NOTE**: Some container matrics like CPU and Memory will not be available for the official version `0.0.7` of kubernetes-app on GKE. For those who want to use that metric you need to deploy Grafana with kubernetes-app plugin version `0.0.8`.

1. Deploy Graphite, make sure you make Carbon port 2003 accessible throughout the cluster with `clusterIP` since the `snap-collector` will not work on Kubernetes DNS, copy the Graphite Carbon service ip address.
2. Deploy Grafana 4 and include kubernets-app plugin. For `0.0.8` version installation, try [here](http://docs.grafana.org/plugins/installation/) with mounted volume, or consult issue [here](https://github.com/grafana/kubernetes-app/issues/29)
3. Add Graphite as datasource in Grafana
4. Enable Kubernetes-app plugin
5. Now you are ready to [Connecting your Cluster](#connecting-to-your-cluster).
6. Obtain your admin http credential by running this command on your local machine that has access to your Kubernetes cluster,

    ```
    gcloud container clusters describe {{Cluster name}} --zone {{Your zone}} --project {{Your project}}
    ```
    
    this command will return yaml with `masterAuth` key which contain username and password to Kubernetes API Server
    
7. In Grafana, add a new cluster using data from step 1 and 4. In `HTTP Setting`, for those who wish to deploy Grafana on the same Kubernetes cluster, you can use `https://kubernetes.default` which is the internal URL to Kubernetes API Server, 

    note the `https`, that is only way to connect to the Kubernetes API Server. 
    
    For those who deploy elsewhere, the Kubernetes API Servier IP Address is included from the return data of step 4. Check on `Basic Auth` and input your credential obtain from step 4. Also check on `Skip TLS Verification (Insecure)` if you do not have any TLS set up for your cluster.
8. On Grafana Write, add the _IP Address_ of your Carbon Host if it's deployed in the same Kubernetes Container. This is the IP you set up on Step 1. The kubernetes DNS Will not work on Snap container so it's better to use IP Address.
9. Deploy and save your setting. to check if this works see if you found `snap` container appear while running

    `kubectl get pods -n kube-system`
    
10. Check if data is feeded to Graphite and Carbon, If there appear to be some data thne your set up must be good to go!


#### Technical Details

Metrics are collected by the [Intel Snap](http://snap-telemetry.io/) collector using the [Docker plugin](https://github.com/intelsdi-x/snap-plugin-collector-docker/blob/master/METRICS.md).  A DaemonSet with Snap is deployed to your Kubernetes cluster when you add a new cluster in the app. For cluster level metrics, one Snap pod is also deployed to the cluster. The [snap_k8s](https://github.com/raintank/snap_k8s) docker image used for this is based off of Intel's Snap docker image.

The following Snap plugins are used to collect metrics:

- [CPU Collector](https://github.com/intelsdi-x/snap-plugin-collector-cpu/blob/master/METRICS.md)
- [Docker Collector](https://github.com/intelsdi-x/snap-plugin-collector-docker/blob/master/METRICS.md)
- [Network Interface Collector](https://github.com/intelsdi-x/snap-plugin-collector-interface/blob/master/METRICS.md)
- [IOStat Collector](https://github.com/intelsdi-x/snap-plugin-collector-iostat)
- [Load Collector](https://github.com/intelsdi-x/snap-plugin-collector-load#collected-metrics)
- [MemInfo Collector](https://github.com/intelsdi-x/snap-plugin-collector-meminfo/blob/master/METRICS.md)
- [Kubestate Collector](https://github.com/raintank/snap-plugin-collector-kubestate)

### Feedback and Questions

Please submit any issues with the app on [Github](https://github.com/raintank/kubernetes-app/issues).

#### Troubleshooting

If there are any problems with metrics not being collected then you can collect some logs with the following steps:

1. Get the snap pod (or pods if you have multiple nodes) name with:

    `kubectl get pods -n kube-system`

2. Check if the task is running with (replace xxxx with the guid):

    `kubectl exec -it snap-xxxxx-n kube-system -- /opt/snap/bin/snaptel task list`

    Include the output in the issue.

3. Get the logs with:

    `kubectl logs snap-xxxxx -n kube-system`

    Include this output in the issue too.
