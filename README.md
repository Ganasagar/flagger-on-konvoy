# Flagger on Konvoy
Flagger helps automate the release process for Kubernetes workloads with a custom resource named canary. It depends on serive-mesh technology to handle advanced deployment strategies, However it can also work with Nginx Ingress controllers however it would be restricted to the a limited number of deployment strategies. In this demo we will walk through installing and configuring on Konvoy Kubernetes cluster   


### Prerequisites

1. Flagger requires a Kubernetes **v1.14** or higher 
2. Konvoy **v1.4** or higher 
3. Istio enbled in Konvoy cluster 
4. Prometheus running as metrics-server in kubeaddons namespace 
5. Privilege access to the Konvoy cluster 



### Install Flagger  

1. #### Add Flagger Helm repository:
```bash
helm repo add flagger https://flagger.app
```

2. #### Install Flagger's Canary CRD:
```bash
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml
```

3. #### Deploy Flagger for Istio: 

```bash
helm upgrade -i flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set meshProvider=istio \
--set metricsServer=http://prometheus-kubeaddons-prom-prometheus.kubeaddons:9090
```
**Note**  Note that Flagger depends on Istio telemetry and Prometheus. When you install Istio with Konvoy the default installation launches using **default profile**. You can also notice that we point Flagger to the prometheus metrics server running in the kubeaddon namespace


Output :
```
Release "flagger" does not exist. Installing it now.
NAME:   flagger
LAST DEPLOYED: Thu Aug 20 14:51:35 2020
NAMESPACE: istio-system
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME     READY  UP-TO-DATE  AVAILABLE  AGE
flagger  0/1    1           0          0s

==> v1/Pod(related)
NAME                      READY  STATUS             RESTARTS  AGE
flagger-58b69886f8-xb69d  0/1    ContainerCreating  0         0s

==> v1/ServiceAccount
NAME     SECRETS  AGE
flagger  1        0s

==> v1beta1/ClusterRole
NAME     AGE
flagger  0s

==> v1beta1/ClusterRoleBinding
NAME     AGE
flagger  0s


NOTES:
Flagger installed
```
4. #### Lets ensure that Flagger was deployed successfully without any challenges 
```bash
kubectl get deploy,po -n istio-system
```
Output:
```
NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/flagger                1/1     1            1           60s
deployment.apps/istio-ingressgateway   2/2     2            2           3d6h
deployment.apps/istio-operator         1/1     1            1           3d6h
deployment.apps/istio-tracing          1/1     1            1           3d6h
deployment.apps/istiod                 2/2     2            2           3d6h
deployment.apps/kiali                  1/1     1            1           3d6h

NAME                                        READY   STATUS      RESTARTS   AGE
pod/flagger-58b69886f8-xb69d                1/1     Running     0          60s
pod/istio-crd-1.6.4-udb5r-gqzcm             0/1     Completed   0          3d6h
pod/istio-ingressgateway-59ddcb9559-bvnwg   1/1     Running     0          3d6h
pod/istio-ingressgateway-59ddcb9559-s9dqx   1/1     Running     0          3d6h
pod/istio-operator-65458b9b8b-wwbg6         1/1     Running     0          3d6h
pod/istio-tracing-dbfcd555-m8h7n            1/1     Running     0          3d6h
pod/istiod-59ddb865bb-8l2c2                 1/1     Running     0          3d6h
pod/istiod-59ddb865bb-qz47z                 1/1     Running     0          3d6h
pod/kiali-6f9b78cdc6-wtmdl                  1/1     Running     0          3d6h
```
Notice that Flagger, Deployment and Pod running successfully. 

5. #### Review logs for Flagger pod to ensure that there are no errors. 
```bash
kubectl logs pod/flagger-58b69886f8-xb69d -n istio-system
```
Output:
```
{"level":"info","ts":"2020-08-20T21:51:36.684Z","caller":"flagger/main.go:112","msg":"Starting flagger version 1.1.0 revision b6d6f32c7f6377e837788db7f7fdb49a25fda057 mesh provider istio"}
{"level":"info","ts":"2020-08-20T21:51:36.699Z","caller":"flagger/main.go:383","msg":"Connected to Kubernetes API v1.17.8"}
{"level":"info","ts":"2020-08-20T21:51:36.699Z","caller":"flagger/main.go:239","msg":"Waiting for canary informer cache to sync"}
{"level":"info","ts":"2020-08-20T21:51:36.799Z","caller":"flagger/main.go:246","msg":"Waiting for metric template informer cache to sync"}
{"level":"info","ts":"2020-08-20T21:51:36.899Z","caller":"flagger/main.go:253","msg":"Waiting for alert provider informer cache to sync"}
{"level":"info","ts":"2020-08-20T21:51:37.007Z","caller":"flagger/main.go:163","msg":"Connected to metrics server http://prometheus-kubeaddons-prom-prometheus.kubeaddons:9090"}
{"level":"info","ts":"2020-08-20T21:51:37.008Z","caller":"server/server.go:29","msg":"Starting HTTP server on port 8080"}
{"level":"info","ts":"2020-08-20T21:51:37.008Z","caller":"controller/controller.go:164","msg":"Starting operator"}
{"level":"info","ts":"2020-08-20T21:51:37.008Z","caller":"controller/controller.go:173","msg":"Started operator workers"}
```
**Note** This ensure that Flagger was able to communicate with the Prometheus Metrics server without any issues. This is very important as Flagger depends on the metrics-server to perform most of its functions. If flagger is unable to communicate with metrics-server rest of the demo would not go as intended.


