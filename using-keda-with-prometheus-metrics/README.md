# Auto scaling applications using keda with prometheus custom metrics


## Prerequisites

- Create a Cloud9 IDE, thats where we will be running all of these steps.
- Existing EKS Cluster
- helm v3 is installed
- 

### Install helm CLI

Before we can get started configuring Helm, we’ll need to first install the command line tools that you will interact with. To do this, run the following:

```sh
# install the command line tools
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

# verify the version
helm version --short

# Download the stable repository so we have something to start with
helm repo add stable https://charts.helm.sh/stable

```

### Clone the git repo

```sh
git clone https://github.com/black-mirror-1/eks-autoscaling-custom-metrics.git

cd eks-autoscaling-custom-metrics/using-keda-with-prometheus-metrics
```

### Install [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Deploy Prometheus

Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. Since its inception in 2012, many companies and organizations have adopted Prometheus, and the project has a very active developer and user community. It is now a standalone open source project and maintained independently of any company. Prometheus joined the Cloud Native Computing Foundation in 2016 as the second hosted project, after Kubernetes.


```sh
# add prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

```

First we are going to install Prometheus. In this example, we are primarily going to use the standard configuration, but we do override the storage class. We will use gp2 EBS volumes for simplicity and demonstration purpose. When deploying in production, you would use io1 volumes with desired IOPS and increase the default storage size in the manifests to get better performance. Run the following command:

```sh
kubectl create namespace prometheus

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```

Check if Prometheus components deployed as expected

```sh
kubectl get all -n prometheus
```

```

NAME                                                 READY   STATUS    RESTARTS   AGE
pod/prometheus-alertmanager-868f8db8c4-67j2x         2/2     Running   0          78s
pod/prometheus-kube-state-metrics-6df5d44568-c4tkn   1/1     Running   0          78s
pod/prometheus-node-exporter-dh6f4                   1/1     Running   0          78s
pod/prometheus-node-exporter-v8rd8                   1/1     Running   0          78s
pod/prometheus-node-exporter-vcbjq                   1/1     Running   0          78s
pod/prometheus-pushgateway-759689fbc6-hvjjm          1/1     Running   0          78s
pod/prometheus-server-546c64d959-qxbzd               2/2     Running   0          78s

NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/prometheus-alertmanager         ClusterIP   10.100.38.47     <none>        80/TCP     78s
service/prometheus-kube-state-metrics   ClusterIP   10.100.165.139   <none>        8080/TCP   78s
service/prometheus-node-exporter        ClusterIP   None             <none>        9100/TCP   78s
service/prometheus-pushgateway          ClusterIP   10.100.150.237   <none>        9091/TCP   78s
service/prometheus-server               ClusterIP   10.100.209.224   <none>        80/TCP     78s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   3         3         3       3            3           <none>          78s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-alertmanager         1/1     1            1           78s
deployment.apps/prometheus-kube-state-metrics   1/1     1            1           78s
deployment.apps/prometheus-pushgateway          1/1     1            1           78s
deployment.apps/prometheus-server               1/1     1            1           78s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-alertmanager-868f8db8c4         1         1         1       78s
replicaset.apps/prometheus-kube-state-metrics-6df5d44568   1         1         1       78s
replicaset.apps/prometheus-pushgateway-759689fbc6          1         1         1       78s
replicaset.apps/prometheus-server-546c64d959               1         1         1       78s
```



## Install KEDA Operator

```sh
# add helm repo for keda
helm repo add kedacore https://kedacore.github.io/charts

# create namespace
kubectl create ns keda

# install keda with values at keda-operater/helm-chart
helm install keda kedacore/keda --namespace keda -f keda-operater/helm-chart/values.yaml

# validate the pods are running
kubectl get pods -n keda
```

The output should look like this

```
NAME                                              READY   STATUS    RESTARTS   AGE
keda-operator-6d74df9577-vc6gs                    1/1     Running   0          48s
keda-operator-metrics-apiserver-fc6df469f-vlf9r   1/1     Running   0          48s
```


## Deploy sample application podinfo

We will deploy a sample application [podinfo](https://github.com/stefanprodan/podinfo), is a tiny web application made with Go that showcases best practices of running microservices in Kubernetes.

In this example, we are deploying the application on fargate, lets create a fargate profile

```sh
eksctl create fargateprofile --namespace podinfo-fp --cluster <cluster-name> --region <region>
```

```sh
kubectl apply -f ../podinfo

# validate if all the resources are deployed
kubectl get all -n podinfo-fp
```


```
NAME                          READY   STATUS    RESTARTS   AGE
pod/podinfo-d4d9484cc-xllmm   1/1     Running   0          13h

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/podinfo   ClusterIP   10.100.154.25   <none>        80/TCP    13h

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/podinfo   1/1     1            1           13h

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/podinfo-d4d9484cc   1         1         1       13h

NAME                                                       REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/keda-hpa-podinfo-hpa   Deployment/podinfo   0/1 (avg)   1         10        1          99m
```

We annotated the podinfo application to scrape the prometheus metrics, lets validate if the targets are being scraped.


```sh
#In order to access the Prometheus server URL, we are going to use the kubectl port-forward command to access the application.
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090
```

In your Cloud9 environment, click Tools / Preview / Preview Running Application. Scroll to the end of the URL and append:

```sh
/targets
```

![](media/prometheus-targets.png)


## Setup KEDA ScaledObject

Let’s create the ScaledObject that will scale the podinfo deployment by querying the metrics stored in prometheus.

```sh
kubectl apply -f keda-operater/podinfo-scaledobject.yaml
```
In this example we are using http_requests_total per second rate as the metric for scaling. But, you can replace the query with a relevant you

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: podinfo-hpa
  namespace: podinfo-fp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  pollingInterval:  30 
  cooldownPeriod:   300
  fallback:
    failureThreshold: 3
    replicas: 2
  minReplicaCount: 1        # Optional. Default: 0
  maxReplicaCount: 10       # Optional. Default: 100
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.prometheus
        threshold: '1'
        # Note: query must return a vector/scalar single element response
        query: sum(rate(http_requests_total{namespace="podinfo-fp",pod="podinfo-d4d9484cc-xllmm"}[2m]))
        metricName: http_requests_total_per_second
```


## Generate load and see autoscaling in action

Lets install [Flagger LoadTester](https://github.com/fluxcd/flagger/tree/main/charts/loadtester) to generate some load to our application

```sh
# add flagger helm repo
helm repo add flagger https://flagger.app

# install loadtester
helm upgrade -i flagger-loadtester flagger/loadtester
```

Lets exec into loadtester and generate some load

```sh
# You can exec into the loadtester pod with the following command
kubectl -n demo exec -it loadtester-xxxx-xxxx

# generate additional traffic using hey as shown below
hey -z 10m -c 5 -q 5 -disable-keepalive http://podinfo.podinfo-fp
```

Describe the hpa to see it in action

```sh
kubectl describe hpa keda-hpa-podinfo -n podinfo-fp
```

```
Events:
  Type    Reason             Age                 From                       Message
  ----    ------             ----                ----                       -------
  Normal  SuccessfulRescale  74s (x2 over 137m)  horizontal-pod-autoscaler  New size: 4; reason: external metric s0-prometheus-http_requests_total_per_second(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: podinfo-hpa,},MatchExpressions:[]LabelSelectorRequirement{},}) above target
  Normal  SuccessfulRescale  59s (x2 over 137m)  horizontal-pod-autoscaler  New size: 8; reason: external metric s0-prometheus-http_requests_total_per_second(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: podinfo-hpa,},MatchExpressions:[]LabelSelectorRequirement{},}) above target
  Normal  SuccessfulRescale  44s (x2 over 136m)  horizontal-pod-autoscaler  New size: 10; reason: external metric s0-prometheus-http_requests_total_per_second(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: podinfo-hpa,},MatchExpressions:[]LabelSelectorRequirement{},}) above target
```


## Cleanup

