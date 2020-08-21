# Flagger on Konvoy
Flagger helps automate the release process for Kubernetes workloads with a custom resource named canary. It depends on serive-mesh technology to handle advanced deployment strategies, However it can also work with Nginx Ingress controllers however it would be restricted to the a limited number of deployment strategies. In this demo we will walk through installing and configuring on Konvoy Kubernetes cluster   


### Prerequisites

1. Flagger requires a Kubernetes **v1.14** or higher 
2. Konvoy **v1.4** or higher 
3. Istio enbled in Konvoy cluster 
4. Prometheus running as metrics-server in kubeaddons namespace 
5. Privilege access to the Konvoy cluster 

![flagger canary](../images/flagger-overview.png)

### Install Flagger  

1. #### Modify the `cluster.yaml` file so that flagger is enable by changing it to true from false 

```bash
......
        - name: flagger
          enabled: true    <--- change this line to true from the default of false
```

2. #### Install Flagger by using Konvoy  
```bash
konvoy deploy addons --yes
```
This would review all the enabled addons and deploy them as necessary. 

Output:
```

STAGE [Deploying Enabled Addons]
external-dns                                                           [OK]
dashboard                                                              [OK]
reloader                                                               [OK]
konvoyconfig                                                           [OK]
opsportal                                                              [OK]
cert-manager                                                           [OK]
gatekeeper                                                             [OK]
defaultstorageclass-protection                                         [OK]
traefik                                                                [OK]
awsebscsiprovisioner                                                   [OK]
istio                                                                  [OK]
dex                                                                    [OK]
kube-oidc-proxy                                                        [OK]
traefik-forward-auth                                                   [OK]
dex-k8s-authenticator                                                  [OK]
prometheus                                                             [OK]
prometheusadapter                                                      [OK]
velero                                                                 [OK]
elasticsearch                                                          [OK]
elasticsearch-curator                                                  [OK]
fluentbit                                                              [OK]
elasticsearchexporter                                                  [OK]
kibana                                                                 [OK]
kommander                                                              [OK]

Kubernetes cluster and addons deployed successfully!
```

3. #### Lets ensure that Flagger was deployed successfully without any challenges 
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

4. #### Review logs for Flagger pod to ensure that there are no errors. 
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
