# Tea lab with traffic splitting

## Introduction

In this lab we are going to deploy the [Tea application and service](tea.yaml) to the default namespace. This Tea application actually creates three deployments, called **tetley** ,**redrose** and **yorkshire** three tea brands. We are going to setup traffic splitting based on host header values or the absence of a host header value. The gateway object and HTTPRoute will be deployed to the default namespace. This is a task that might be completed by the **Applications Developer**.

Copy and paste the following code snippet to deploy the tea applications and services.

```yaml
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redrose
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redrose
  template:
    metadata:
      labels:
        app: redrose
    spec:
      containers:
      - name: redrose
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: redrose
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: redrose
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tetley
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tetley
  template:
    metadata:
      labels:
        app: tetley
    spec:
      containers:
      - name: tetley
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tetley
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: tetley
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yorkshire
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yorkshire
  template:
    metadata:
      labels:
        app: yorkshire
    spec:
      containers:
      - name: yorkshire
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: yorkshire
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: yorkshire
EOF
```

Now check your new pods and services in the **default** namespace.

```bash
kubectl get pod,svc -owide
```

<details>
  <summary><h3>Example output</h3></summary>

  ```bash
  NAME                             READY   STATUS    RESTARTS   AGE   IP              NODE                    NOMINATED NODE   READINESS GATES
pod/redrose-5f56dfb684-pbx6p     1/1     Running   0          8s    10.244.67.153   w1-mgmt.lab.f5npi.net   <none>           <none>
pod/tetley-65b9df8768-zm2mw      1/1     Running   0          8s    10.244.67.154   w1-mgmt.lab.f5npi.net   <none>           <none>
pod/yorkshire-7f84486bf6-ggsvc   1/1     Running   0          8s    10.244.82.151   w2-mgmt.lab.f5npi.net   <none>           <none>

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   9d    <none>
service/redrose      ClusterIP   10.107.126.10   <none>        80/TCP    8s    app=redrose
service/tetley       ClusterIP   10.99.196.81    <none>        80/TCP    8s    app=tetley
service/yorkshire    ClusterIP   10.97.91.223    <none>        80/TCP    8s    app=yorkshire

  ```
</details>

Then we will create an [NGINX Gateway Fabric gateway](gateway.yaml) object that will enable traffic into the default namespace based on the following criteria.

| Property      | Values                 |
| ------------- | ---------------------- |
| port          | `80`                   |
| protocol      | `HTTP`                 |
| hostname      | `tea.lab.f5npi.net` |

This could be the responsibility of the **cluster administrator**.

Copy and paste the following code snippet to deploy the coffee gateway.

```yaml
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cafe-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: tea.lab.f5npi.net
EOF
```

Check the new cafe-gateway health.

```bash
kubectl get gateways cafe-gateway
kubectl describe gateways cafe-gateway
```
<details>
  <summary><h3>Example output</h3></summary>

  ```bash
  NAME           CLASS   ADDRESS       PROGRAMMED   AGE
cafe-gateway   nginx   10.1.10.100   True         9s
Name:         cafe-gateway
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         Gateway
Metadata:
  Creation Timestamp:  2024-07-15T21:33:05Z
  Generation:          1
  Resource Version:    125462
  UID:                 9b835a17-efe6-4405-a0c5-24ac1bd45c42
Spec:
  Gateway Class Name:  nginx
  Listeners:
    Allowed Routes:
      Namespaces:
        From:  Same
    Hostname:  tea.lab.f5npi.net
    Name:      http
    Port:      80
    Protocol:  HTTP
Status:
  Addresses:
    Type:   IPAddress
    Value:  10.1.10.100
  Conditions:
    Last Transition Time:  2024-07-15T21:33:05Z
    Message:               Gateway is accepted
    Observed Generation:   1
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
    Last Transition Time:  2024-07-15T21:33:05Z
    Message:               Gateway is programmed
    Observed Generation:   1
    Reason:                Programmed
    Status:                True
    Type:                  Programmed
  Listeners:
    Attached Routes:  0
    Conditions:
      Last Transition Time:  2024-07-15T21:33:05Z
      Message:               Listener is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-15T21:33:05Z
      Message:               Listener is programmed
      Observed Generation:   1
      Reason:                Programmed
      Status:                True
      Type:                  Programmed
      Last Transition Time:  2024-07-15T21:33:05Z
      Message:               All references are resolved
      Observed Generation:   1
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
      Last Transition Time:  2024-07-15T21:33:05Z
      Message:               No conflicts
      Observed Generation:   1
      Reason:                NoConflicts
      Status:                False
      Type:                  Conflicted
    Name:                    http
    Supported Kinds:
      Group:  gateway.networking.k8s.io
      Kind:   HTTPRoute
Events:       <none>
  ```
</details>

Next we will create the [HTTPRoute](two-tea-brands-httpRoute.yaml) that will traffic splitting based on header values. We are using the Exact header so the header values must be an exact match. Alternatively we could have used Prefix. This httpRoute will route request that lack a host header value to redrose by default.  Request that include the host header value **brand:tetley** will be routed to the tetley application.  This could be the responsibility of the **Application Developer**.

>**Note**: See the [HTTPRouteRule spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.HTTPRouteRule) for more details.

Copy and paste the following code snippet to create the HTTPRoute for the tea application called `httpRoute-tea.yaml`.

```yaml
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: tea-httproute
spec:
  parentRefs:
  - name: cafe-gateway
    sectionName: http
  hostnames:
  - "tea.lab.f5npi.net"
  rules:
  - matches:
    - headers:
      - type: Exact
        name: brands
        value: tetley
    backendRefs:
    - name: tetley
      port: 80
  - backendRefs:
    - name: redrose
      port: 80 " > httpRoute-tea.yaml
EOF
```

Now you can create this object and validate it is healthy.

```bash
kubectl create -f httpRoute-tea.yaml
kubectl get httproutes tea-httproute
kubectl describe httproutes tea-httproute
```

<details>
  <summary><h3>Example output</h3></summary>

  ```bash
  f5admin@bastion:~$ kubectl create -f httpRoute-tea.yaml
httproute.gateway.networking.k8s.io/tea-httproute created

f5admin@bastion:~$ kubectl get httproutes tea-httproute
NAME            HOSTNAMES               AGE
tea-httproute   ["tea.lab.f5npi.net"]   35s

f5admin@bastion:~$ kubectl describe httproutes tea-httproute
Name:         tea-httproute
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         HTTPRoute
Metadata:
  Creation Timestamp:  2024-07-15T21:34:52Z
  Generation:          1
  Resource Version:    125747
  UID:                 b0a60341-ea4c-4cb5-a93f-4de3a1c4b7bb
Spec:
  Hostnames:
    tea.lab.f5npi.net
  Parent Refs:
    Group:         gateway.networking.k8s.io
    Kind:          Gateway
    Name:          cafe-gateway
    Section Name:  http
  Rules:
    Backend Refs:
      Group:
      Kind:    Service
      Name:    tetley
      Port:    80
      Weight:  1
    Matches:
      Headers:
        Name:   brands
        Type:   Exact
        Value:  tetley
      Path:
        Type:   PathPrefix
        Value:  /
    Backend Refs:
      Group:
      Kind:    Service
      Name:    redrose
      Port:    80
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /
Status:
  Parents:
    Conditions:
      Last Transition Time:  2024-07-15T21:34:53Z
      Message:               The route is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-15T21:34:53Z
      Message:               All references are resolved
      Observed Generation:   1
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
    Parent Ref:
      Group:         gateway.networking.k8s.io
      Kind:          Gateway
      Name:          cafe-gateway
      Namespace:     default
      Section Name:  http
Events:              <none>

  ```
</details>

## Testing

Now test your newly exposed application using the **NGINX Gateway Fabric HTTPRoute** we just deployed that is linked to the **cafe-gateway** object.  We should find that generic requests to tea.lab.f5npi.net/tea are routed to the redrose tea applications but for requests that in include the header values **brands:tetley** the request is routed to the tetley tea application. However, when you test using the header **brands:yorkshire** you are directed to the redrose application.

Your challenge it to update the httpRoute to direct requests with the header **brands:yorkshire** to the yorkshire tea application.

Here are some sample curl test urls, the first two should work but the request for with the header **brand:yorkshire** should fail until you update the httpRoute.

```bash
curl tea.lab.f5npi.net/tea
curl -H "brands:tetley" tea.lab.f5npi.net/tea
curl -H "brands:yorkshire" tea.lab.f5npi.net/tea
```

## Second lab objective

The cafe department now wants to enable a basic canary configuration that will route traffic for tea.lab.f5npi.net/brew to **redrose** for 90% of the requests but send 10% of their requests to **yorkshire**.

The following code block will create a file called `httpRoute-tea-weight.yaml`.  It their current configuration which results in only seeing traffic routed to **redrose**. You need to update the configuration to support their weighting request of 90% directed to **redrose** and 10% of the requests being routed to **yorkshire**.

>**Question**: What weight value is assigned to a service if you don't provide value?  What does a weight value of 0 do?

```yaml
echo "apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: tea-weight-httproute
spec:
  parentRefs:
  - name: cafe-gateway
    sectionName: http
  hostnames:
  - "tea.lab.f5npi.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /brew
    backendRefs:
    - name: redrose
      port: 80" > httpRoute-tea-weight.yaml
```

Now you can create this object and validate it is healthy.

```bash
kubectl create -f httpRoute-tea-weight.yaml
kubectl get httproutes tea-weight-httproute
kubectl describe httproutes tea-weight-httproute
```

<details>
  <summary><h3>Example output (Not a solution!)</h3></summary>

  ```bash
f5admin@bastion:~$ kubectl create -f httpRoute-tea-weight.yaml
httproute.gateway.networking.k8s.io/tea-weight-httproute created

f5admin@bastion:~$ kubectl get httproutes tea-weight-httproute
NAME                   HOSTNAMES               AGE
tea-weight-httproute   ["tea.lab.f5npi.net"]   76s

f5admin@bastion:~$ kubectl describe httproutes tea-weight-httproute
Name:         tea-weight-httproute
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         HTTPRoute
Metadata:
  Creation Timestamp:  2024-07-15T21:53:22Z
  Generation:          1
  Resource Version:    128788
  UID:                 8a745962-d889-47d1-b34b-83400e7443cf
Spec:
  Hostnames:
    tea.lab.f5npi.net
  Parent Refs:
    Group:         gateway.networking.k8s.io
    Kind:          Gateway
    Name:          cafe-gateway
    Section Name:  http
  Rules:
    Backend Refs:
      Group:
      Kind:    Service
      Name:    redrose
      Port:    80
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /brew
Status:
  Parents:
    Conditions:
      Last Transition Time:  2024-07-15T21:53:22Z
      Message:               The route is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-15T21:53:22Z
      Message:               All references are resolved
      Observed Generation:   1
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
    Parent Ref:
      Group:         gateway.networking.k8s.io
      Kind:          Gateway
      Name:          cafe-gateway
      Namespace:     default
      Section Name:  http
Events:              <none>
  ```
</details>

Now work on updating deployment to support the request and test your solution.

Test your configuration using.

```bash
curl tea.lab.f5npi.net/brew
```
<details>
  <summary><h3>Example output</h3></summary>

  ```bash
  f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.155:8080
Server name: redrose-5f56dfb684-56sf9
Date: 15/Jul/2024:21:59:08 +0000
URI: /brew
Request ID: 77cb6a3786f1abc9ea20160ea45d3646
f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.156:8080
Server name: yorkshire-7f84486bf6-9qqcj
Date: 15/Jul/2024:21:59:10 +0000
URI: /brew
Request ID: f44ed940ee41f80643898db75059d736
f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.155:8080
Server name: redrose-5f56dfb684-56sf9
Date: 15/Jul/2024:21:59:11 +0000
URI: /brew
Request ID: 7e32c6f4adb324c8a149b9f528fd6fa1
f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.155:8080
Server name: redrose-5f56dfb684-56sf9
Date: 15/Jul/2024:21:59:12 +0000
URI: /brew
Request ID: 21f5de1afb866a2fc938b37b026eeecf
f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.155:8080
Server name: redrose-5f56dfb684-56sf9
Date: 15/Jul/2024:21:59:13 +0000
URI: /brew
Request ID: b2fecc4b7ce0f4c14c4627d2e6ee859a
f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.155:8080
Server name: redrose-5f56dfb684-56sf9
Date: 15/Jul/2024:21:59:13 +0000
URI: /brew
Request ID: 3d6bb6c376026b93865737963d3adb1a
f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.155:8080
Server name: redrose-5f56dfb684-56sf9
Date: 15/Jul/2024:21:59:15 +0000
URI: /brew
Request ID: 700db96a6190b4176b394d74987e3370
f5admin@bastion:~$ curl tea.lab.f5npi.net/brew
Server address: 10.244.67.155:8080
Server name: redrose-5f56dfb684-56sf9
Date: 15/Jul/2024:21:59:16 +0000
URI: /brew
Request ID: 7d213c8bd3f8e12c45714dc3c6dd08cf
  ```
</details>

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments.apps redrose tetley yorkshire
kubectl delete svc redrose tetley yorkshire
kubectl delete gateways cafe-gateway
kubectl delete httproutes tea-httproute
kubectl delete httproutes tea-weight-httproute
```

>**Note**: If you would prefer to manually create and deploy the configurations you can find the source YAML files here:

- [Tea applications and services](tea-brands.yaml)
- [Cafe Gateway](cafe-gateway.yaml)
- [Tea http Route that supports redrose and tetley](two-tea-brands-httpRoute.yaml)

### httpRoute solutions

- [All tea brand http Route solution using headers](all-tea-brands-httpRoute.yaml)
- [Routing traffic based on weights](httpRoute-tea-weight.yaml)

Previous: [Demo](../demo/README.md)

Next: [Fix It Lab](../fixit/README.md)

---

## End of lab
