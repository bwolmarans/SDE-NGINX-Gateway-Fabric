# Advanced Routing Lab

This lab is intended to be completed by individuals or small groups during the Up skilling sessions but it maybe completed at anytime.

## Introduction

In this lab, we deploy NGF in this lab scenario:
https://docs.nginx.com/nginx-gateway-fabric/how-to/traffic-management/advanced-routing/

The hostnames are changed from ```.example.com``` to ```.lab.f5npi.net``` to match the lab environment.


## Lab Procedure

Before you begin use the appropriate ```kubectl``` commands to delete any:

* deployments
* services (besides kubernetes)
* httproutes
* configmaps
* pods
* gateways


>**Question**: Were there any of these kind of objects? How did you remove them?


Copy and paste the following code snippet to deploy the app.

```bash
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: coffee-v1
  template:
    metadata:
      labels:
        app: coffee-v1
    spec:
      containers:
      - name: coffee-v1
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-v1-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: coffee-v1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: coffee-v2
  template:
    metadata:
      labels:
        app: coffee-v2
    spec:
      containers:
      - name: coffee-v2
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-v2-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: coffee-v2
EOF
```



<details><summary>Example</summary>

```
deployment.apps/coffee-v1 created
service/coffee-v1-svc created
deployment.apps/coffee-v2 created
service/coffee-v2-svc created
```

</details>



Now check your new pods and services.

```bash
kubectl get pod,svc -owide
```

```bash
kubectl get deployments
kubectl describe deployments
```

<details><summary>Example</summary>

```
root@bastion:/# kubectl get deployments.apps 
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
coffee-v1   1/1     1            1           2m52s
coffee-v2   1/1     1            1           2m52s

root@bastion:/# kubectl get deployments.apps 
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
coffee-v1   1/1     1            1           2m52s
coffee-v2   1/1     1            1           2m52s

root@bastion:/# k describe deployments
Name:                   coffee-v1
Namespace:              default
CreationTimestamp:      Tue, 16 Jul 2024 04:31:16 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=coffee-v1
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=coffee-v1
  Containers:
   coffee-v1:
    Image:         nginxdemos/nginx-hello:plain-text
    Port:          8080/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   coffee-v1-76c7c85bbd (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  4m1s  deployment-controller  Scaled up replica set coffee-v1-76c7c85bbd to 1


Name:                   coffee-v2
Namespace:              default
CreationTimestamp:      Tue, 6 Jul 2024 04:31:16 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=coffee-v2
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=coffee-v2
  Containers:
   coffee-v2:
    Image:         nginxdemos/nginx-hello:plain-text
    Port:          8080/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   coffee-v2-7d47fc86cb (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  4m1s  deployment-controller  Scaled up replica set coffee-v2-7d47fc86cb to 1
```

>**Question**: How many pods and services do you see?


--------------


Next, create an [NGINX Gateway Fabric gateway](post-gateway.yaml) object that will enable traffic into the default namespace based on the following criteria.

- port: 80
- protocol: HTTP
- hostname: "*.lab.f5npi.net"

This could be the responsibility of the **cluster administrator**.

Copy and paste the following code snippet to deploy the gateway.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cafe
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: "*.lab.f5npi.net"
EOF
```


&nbsp;


Next, check the new post-n-get-gateway health.




</details>


Next create the [HTTPRoute](httproute.yaml) that will allow traffic inbound to our application and leverage the NGINX Gateway Fabric gateway object.


```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: coffee
spec:
  parentRefs:
  - name: cafe
    sectionName: http
  hostnames:
  - cafe.lab.f5npi.net
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /coffee
    backendRefs:
    - name: coffee-v1-svc
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /coffee
      headers:
      - name: version
        value: v2
    - path:
        type: PathPrefix
        value: /coffee
      queryParams:
      - name: TEST
        value: v2
    backendRefs:
    - name: coffee-v2-svc
      port: 80
EOF
```



Now check on the status of the HTTPRoute.

```bash
kubectl get httproute
kubectl describe httproute
```

<details><summary>Example</summary>

```

root@bastion:/# k get httproute
NAME     HOSTNAMES                AGE
coffee   ["cafe.lab.f5npi.net"]   56s
root@bastion:/# k describe httproute
Name:         coffee
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         HTTPRoute
Metadata:
  Creation Timestamp:  2024-07-16T04:52:31Z
  Generation:          1
  Resource Version:    189653
  UID:                 8ce9afe6-31cb-48c9-b02e-dae26fb53f92
Spec:
  Hostnames:
    cafe.lab.f5npi.net
  Parent Refs:
    Group:         gateway.networking.k8s.io
    Kind:          Gateway
    Name:          cafe
    Section Name:  http
  Rules:
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    coffee-v1-svc
      Port:    80
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /coffee
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    coffee-v2-svc
      Port:    80
      Weight:  1
    Matches:
      Headers:
        Name:   version
        Type:   Exact
        Value:  v2
      Path:
        Type:   PathPrefix
        Value:  /coffee
      Path:
        Type:   PathPrefix
        Value:  /coffee
      Query Params:
        Name:   TEST
        Type:   Exact
        Value:  v2
Status:
  Parents:
    Conditions:
      Last Transition Time:  2024-07-16T04:52:32Z
      Message:               The route is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-16T04:52:32Z
      Message:               All references are resolved
      Observed Generation:   1
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
    Parent Ref:
      Group:         gateway.networking.k8s.io
      Kind:          Gateway
      Name:          cafe
      Namespace:     default
      Section Name:  http
Events:              <none>

```

</details>

## Testing

Now test your newly exposed application using the **NGINX Gateway Fabric HTTPRoute** we just deployed and that is linked the **cafe-gateway** object.

```bash
curl cafe.lab.f5npi.net/coffee
curl cafe.lab.f5npi.net/coffee -H "version:v2"
curl cafe.lab.f5npi.net/coffee?TEST=v2
```


Look at ```/etc/hosts``` and find some other hostnames that resolve to the same address. 


>**Question**: For these 3 requests, which HTTPRoute rules match which request? Why?

<details><summary>Solution</summary>

```bash
curl cafe.lab.f5npi.net/coffee
```
This matches the pathPrefix /coffee
 ```bash
curl cafe.lab.f5npi.net/coffee -H "version:v2"
```
This matches the headers "version"

```bash
curl cafe.lab.f5npi.net/coffee?TEST=v2
```
This matches the queryParams "TEST"

</details>

## Create the Tea Applications

```bash
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tea-post
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tea-post
  template:
    metadata:
      labels:
        app: tea-post
    spec:
      containers:
      - name: tea-post
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tea-post-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: tea-post
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tea
  template:
    metadata:
      labels:
        app: tea
    spec:
      containers:
      - name: tea
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tea-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: tea
EOF
```


Deploy the HTTPRoute for the Tea application:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: tea
spec:
  parentRefs:
  - name: cafe
  hostnames:
  - cafe.lab.f5npi.net
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /tea
      method: POST
    backendRefs:
    - name: tea-post-svc
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /tea
      method: GET
    backendRefs:
    - name: tea-svc
      port: 80
EOF
```

Next, perform web requests:


```bash
curl cafe.lab.f5npi.net/tea
curl cafe.lab.f5npi.net/tea -X POST
curl cafe.lab.f5npi.net/tea -X OPTIONS
```

>**Question**: For these 3 requests, which HTTPRoute rules match which request? Why?

<details><summary>Solution</summary>

```bash
curl cafe.lab.f5npi.net/tea
```
This matches that pathPrefix /tea
 ```bash
curl cafe.lab.f5npi.net/tea -X POST
```
This matches the method POST
```bash
curl cafe.lab.f5npi.net/tea -X OPTIONS
```
This gets a 404 because it matches nothing. There is no rule looking for the method "OPTIONS". NGF does not find any resource to handle the request and give a 404 Not Found.

</details>

------------------

>**Note**: If you would prefer to manually create and deploy the configurations you can find the source YAML files:

- [application and service](coffee.yaml)
- [Gateway](gateway.yaml)
- [HTTPRoute](httproute.yaml)
- [HTTPRoute](tea.yaml)


## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployment/tea deployment/coffee httproute/tea httproute/coffee
```

Previous: [Demo](../demo/README.md)

Next: [Fix It Lab](../fixit/README.md)

---

## End of lab
