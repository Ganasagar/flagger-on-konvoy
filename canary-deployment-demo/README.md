### Prerequisites

1. Flagger requires a Kubernetes **v1.14** or higher 
2. Konvoy **v1.4** or higher 
3. Istio enbled in Konvoy cluster 
4. Prometheus running as metrics-server in kubeaddons namespace 
5. Privilege access to the Konvoy cluster 
6. Flagger is installaled as per previous section

#### Create an ingress gateway to expose the demo app outside of the mesh:
```
kubectl apply -f gateway.yaml
```



### Bootstrap 
Flagger takes a Kubernetes deployment and optionally a horizontal pod autoscaler (HPA), then creates a series of objects (Kubernetes deployments, ClusterIP services, Istio destination rules and virtual services). These objects expose the application inside the mesh and drive the canary analysis and promotion.

![flagger canary](images/flagger-canary.png)

1. #### Create a test namespace for the demo
```bash
kubectl create ns test
```

2. ####  Enable Istio sidecar injection for the namespace created.

```bash
kubectl label namespace test istio-injection=enabled
```
**Note** This ensures that Istio injects sidecar proxies onto each and every pod deployed in the namespace. 

3. #### Create a deployment and a horizontal pod autoscaler:
```
kubectl apply -k github.com/weaveworks/flagger//kustomize/podinfo
```
4. #### Deploy the load testing service to generate traffic during the canary analysis:
```
kubectl apply -k github.com/weaveworks/flagger//kustomize/tester
```
5. #### Create a canary custom resource (replace example.com with your own domain):
```
kubectl apply -f poinfo-canary-spec.yaml
```
**Note:** This file is the key resource specification. You define all the characteristics of the canary deployment in this file. 


### Automated canary promotion

#### 1. Trigger a canary deployment by updating the container image:
```bash
kubectl -n test set image deployment/podinfo \
podinfod=stefanprodan/podinfo:3.1.1
```

#### 2. Flagger detects that the deployment revision changed and starts a new rollout:
```
kubectl -n test describe canary/podinfo
```

Output:
```
Status:
  Canary Weight:         0
  Failed Checks:         0
  Phase:                 Succeeded
Events:
  Type     Reason  Age   From     Message
  ----     ------  ----  ----     -------
  Normal   Synced  3m    flagger  New revision detected podinfo.test
  Normal   Synced  3m    flagger  Scaling up podinfo.test
  Warning  Synced  3m    flagger  Waiting for podinfo.test rollout to finish: 0 of 1 updated replicas are available
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 5
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 10
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 15
  Normal   Synced  2m    flagger  Advance podinfo.test canary weight 20
  Normal   Synced  2m    flagger  Advance podinfo.test canary weight 25
  Normal   Synced  1m    flagger  Advance podinfo.test canary weight 30
  Normal   Synced  1m    flagger  Advance podinfo.test canary weight 35
  Normal   Synced  55s   flagger  Advance podinfo.test canary weight 40
  Normal   Synced  45s   flagger  Advance podinfo.test canary weight 45
  Normal   Synced  35s   flagger  Advance podinfo.test canary weight 50
  Normal   Synced  25s   flagger  Copying podinfo.test template spec to podinfo-primary.test
  Warning  Synced  15s   flagger  Waiting for podinfo-primary.test rollout to finish: 1 of 2 updated replicas are available
  Normal   Synced  5s    flagger  Promotion completed! Scaling down podinfo.test
```
**Note** that if you apply new changes to the deployment during the canary analysis, Flagger will restart the analysis.

A canary deployment is triggered by changes in any of the following objects:
 1. Deployment PodSpec (container image, command, ports, env, resources, etc)
 2. ConfigMaps mounted as volumes or mapped to environment variables
 3. Secrets mounted as volumes or mapped to environment variables

#### 3. You can monitor all canaries with:
```
watch kubectl get canaries --all-namespaces
```

Output:
```
NAMESPACE   NAME      STATUS        WEIGHT   LASTTRANSITIONTIME
test        podinfo   Progressing   15       2019-01-16T14:05:07Z
prod        frontend  Succeeded     0        2019-01-15T16:15:07Z
prod        backend   Failed        0        2019-01-14T17:05:07Z
```

### Automated rollback 

During the canary analysis you can generate HTTP 500 errors and high latency to test if Flagger pauses the rollout. We will leverage the loadtester application to simulate load and errors 

#### 1. Trigger another canary deployment:
```
kubectl -n test set image deployment/podinfo \
podinfod=stefanprodan/podinfo:3.1.2
```

#### 2. Exec into the load tester pod with:
```
kubectl -n test exec -it flagger-loadtester-xx-xx sh
```

#### 3. Generate HTTP 500 errors:
```bash
watch curl http://podinfo-canary:9898/status/500
```
#### 4. Generate latency:
```
watch curl http://podinfo-canary:9898/delay/1
```
When the number of failed checks reaches the canary analysis threshold, the traffic is routed back to the primary, the canary is scaled to zero and the rollout is marked as failed
```
kubectl -n test describe canary/podinfo

Status:
  Canary Weight:         0
  Failed Checks:         10
  Phase:                 Failed
Events:
  Type     Reason  Age   From     Message
  ----     ------  ----  ----     -------
  Normal   Synced  3m    flagger  Starting canary deployment for podinfo.test
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 5
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 10
  Normal   Synced  3m    flagger  Advance podinfo.test canary weight 15
  Normal   Synced  3m    flagger  Halt podinfo.test advancement success rate 69.17% < 99%
  Normal   Synced  2m    flagger  Halt podinfo.test advancement success rate 61.39% < 99%
  Normal   Synced  2m    flagger  Halt podinfo.test advancement success rate 55.06% < 99%
  Normal   Synced  2m    flagger  Halt podinfo.test advancement success rate 47.00% < 99%
  Normal   Synced  2m    flagger  (combined from similar events): Halt podinfo.test advancement success rate 38.08% < 99%
  Warning  Synced  1m    flagger  Rolling back podinfo.test failed checks threshold reached 10
  Warning  Synced  1m    flagger  Canary failed! Scaling down podinfo.test
```

