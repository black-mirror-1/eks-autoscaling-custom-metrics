# Auto scaling applications using keda with prometheus custom metrics


## Prerequisites

- Create a Cloud9 IDE, thats where we will be running all of these steps.
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux)
- Install [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- Install/Upgrade [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) to version 2


### Install helm CLI

Helm is a package manager for Kubernetes that packages multiple Kubernetes resources into a single logical deployment unit called a Chart. Charts are easy to create, version, share, and publish.

We’ll install the command line tools that you will interact with to get started on helm.

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

### Create an EKS cluster

Create an EKS cluster in `us-east-2` with a fargate profile for `kube-system` to install the EKS add-ons `vpc-cni`, `coredns` and `kube-proxy` and a managed nodegroup.

```sh
eksctl create cluster -f eks-cluster/eksctl-config.yaml
```

The cluster can take about 10-15 min. Lets test if we are able to connect to the cluster.

```sh
kubectl get nodes
```

### Install Metrics Server

[Metrics Server](https://github.com/kubernetes-sigs/metrics-server) is a scalable, efficient source of container resource metrics for Kubernetes built-in autoscaling pipelines using metrics such as CPU/Memeory. We will extend the capability to scale based on customer metrics using KEDA Operater.
```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Deploy Prometheus

[Prometheus](https://prometheus.io/) is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. We will use prometheus to monitor our sample application and use a http metric from prometheus to scale the application.


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
kubectl get deploy,ds -n prometheus
```

```
NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-alertmanager         1/1     1            1           4m38s
deployment.apps/prometheus-kube-state-metrics   1/1     1            1           4m38s
deployment.apps/prometheus-pushgateway          1/1     1            1           4m38s
deployment.apps/prometheus-server               1/1     1            1           4m38s

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   2         2         2       2            2           <none>          4m38s

```

```sh
#In order to access the Prometheus server URL, we are going to use the kubectl port-forward command to access the application.
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090
```

In your Cloud9 environment, click Tools / Preview / Preview Running Application. Scroll to the end of the URL and append:

```sh
/targets
```



## Install KEDA Operator

[KEDA](https://keda.sh/) is a single-purpose and lightweight component that can be added into any Kubernetes cluster. KEDA works alongside standard Kubernetes components like the Horizontal Pod Autoscaler and can extend functionality without overwriting or duplication

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

We will deploy a sample application [podinfo](https://github.com/stefanprodan/podinfo), a tiny web application made with Go that showcases best practices of running microservices in Kubernetes.

In this example, we are deploying the application on fargate, lets create a fargate profile

```sh
eksctl create fargateprofile podinfo-profile --namespace podinfo-fp --cluster eks-demo --region us-east-2
```

```sh
kubectl apply -f ../podinfo/deployment.yaml

# validate if the deployment is ready
kubectl get deploy -n podinfo-fp
```


```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
podinfo   1/1     1            1           89s        99m
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

![](/media/prometheus-targets.png)



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
        query: sum(rate(http_requests_total{app="podinfo"}[2m]))
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
kubectl exec -i -t $(kubectl get po -l app=loadtester --output=jsonpath={.items..metadata.name}) -- sh

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

```sh
helm uninstall flagger-loadtester
helm uninstall prometheus -n prometheus
helm uninstall keda -n keda 

# KEDA ScaledObject 
kubectl delete -f keda-operater/podinfo-scaledobject.yaml

# podinfo sample-app
kubectl delete -f ../podinfo/deployment.yaml

# Metrics Server
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# delete the EKS cluster
eksctl delete cluster --name=eks-demo --region us-east-2
```