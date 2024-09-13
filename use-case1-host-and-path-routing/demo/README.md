# Coffee Demo

This lab is intended for the instruction to demonstrate this use case using the instructions from
this page. Users can either watch or following along using their own live environment.

## Introduction

In this demo we are going to deploy the coffee application and service to the default namespace.  This would might be completed by the **Applications Developer**.

## Demo

Copy and paste the following code snippet to deploy the coffee application and service.

> If you prefer to manually create and deploy this application, click [here](coffee.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee
spec:
  replicas: 1
  selector:
    matchLabels:
      app: coffee
  template:
    metadata:
      labels:
        app: coffee
    spec:
      containers:
      - name: coffee
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: coffee
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: coffee
EOF
```

Now check your new pod and service

```bash
kubectl get pod,svc -owide
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  $ kubectl get pod,svc -owide
  NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE                    NOMINATED NODE   READINESS GATES
  pod/coffee-6b8b6d6486-pclbw   1/1     Running   0          10s   10.244.191.142   w3-mgmt.lab.f5npi.net   <none>           <none>

  NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
  service/coffee       ClusterIP   10.101.152.33   <none>        80/TCP    10s     app=coffee
  service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   4d22h   <none>
  ```

</details>

Then we will create an [NGINX Gateway Fabric gateway](gateway.yaml) object that will enable traffic into the default namespace based on the following criteria.

| Property      | Values                 |
| ------------- | ---------------------- |
| port          | `80`                   |
| protocol      | `HTTP`                 |
| hostname      | `coffee.lab.f5npi.net` |

This could be the responsibility of the **Cluster Administrator**.

Copy and paste the following code snippet to deploy the coffee gateway.

> If you prefer to manually create and deploy this gateway, click [here](gateway.yaml) for the
> YAML file.

```bash
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
    hostname: coffee.lab.f5npi.net
EOF
```

Check the new cafe-gateway health.

```bash
kubectl get gateways cafe-gateway
kubectl describe gateways cafe-gateway
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  $ kubectl get gateways cafe-gateway
  NAME           CLASS   ADDRESS       PROGRAMMED   AGE
  cafe-gateway   nginx   10.1.10.100   True         8s

  $ kubectl describe gateways cafe-gateway
  Name:         cafe-gateway
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         Gateway
  Metadata:
    Creation Timestamp:  2024-07-10T21:54:15Z
    Generation:          1
    Resource Version:    110298
    UID:                 470ec269-71a2-4002-b327-4729ef3a43f7
  Spec:
    Gateway Class Name:  nginx
    Listeners:
      Allowed Routes:
        Namespaces:
          From:  Same
      Hostname:  coffee.lab.f5npi.net
      Name:      http
      Port:      80
      Protocol:  HTTP
  Status:
    Addresses:
      Type:   IPAddress
      Value:  10.1.10.100
    Conditions:
      Last Transition Time:  2024-07-10T21:54:16Z
      Message:               Gateway is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-10T21:54:16Z
      Message:               Gateway is programmed
      Observed Generation:   1
      Reason:                Programmed
      Status:                True
      Type:                  Programmed
    Listeners:
      Attached Routes:  0
      Conditions:
        Last Transition Time:  2024-07-10T21:54:16Z
        Message:               Listener is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-10T21:54:16Z
        Message:               Listener is programmed
        Observed Generation:   1
        Reason:                Programmed
        Status:                True
        Type:                  Programmed
        Last Transition Time:  2024-07-10T21:54:16Z
        Message:               All references are resolved
        Observed Generation:   1
        Reason:                ResolvedRefs
        Status:                True
        Type:                  ResolvedRefs
        Last Transition Time:  2024-07-10T21:54:16Z
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

Next we will create the **HTTPRoute** that will allow traffic inbound to our coffee application and leverage the NGINX Gateway Fabric gateway object.  This object will also define the path uri of **/coffee** and backend service (POD) to route traffic to.  This could be the responsibility of the **Application Developer**.

Copy and paste the following code snippet to deploy the coffee gateway.

> If you prefer to manually create and deploy this HTTPRoute, click [here](httpRoute-coffee.yaml)
> for the YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: coffee
spec:
  parentRefs:
  - name: cafe-gateway
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /coffee
    backendRefs:
    - name: coffee
      port: 80
EOF
```

```bash
kubectl describe httproutes coffee
kubectl get httproutes coffee
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  $ kubectl describe httproutes coffee
  Name:         coffee
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         HTTPRoute
  Metadata:
    Creation Timestamp:  2024-07-10T21:56:11Z
    Generation:          1
    Resource Version:    110605
    UID:                 16cf856d-10b1-4852-9e23-b5c1ba82305b
  Spec:
    Parent Refs:
      Group:         gateway.networking.k8s.io
      Kind:          Gateway
      Name:          cafe-gateway
      Section Name:  http
    Rules:
      Backend Refs:
        Group:
        Kind:    Service
        Name:    coffee
        Port:    80
        Weight:  1
      Matches:
        Path:
          Type:   PathPrefix
          Value:  /coffee
  Status:
    Parents:
      Conditions:
        Last Transition Time:  2024-07-10T21:56:11Z
        Message:               The route is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-10T21:56:11Z
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

  $ kubectl get httproutes coffee
  NAME     HOSTNAMES   AGE
  coffee               13s
  ```

</details>

## Testing

Now test your newly exposed application using the **NGINX Gateway Fabric HTTPRoute** we just deployed that is linked to the **cafe-gateway** object.

```bash
curl coffee.lab.f5npi.net/coffee
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  $ curl coffee.lab.f5npi.net/coffee
  Server address: 10.244.191.142:8080
  Server name: coffee-6b8b6d6486-pclbw
  Date: 10/Jul/2024:21:57:37 +0000
  URI: /coffee
  Request ID: 69e879504432b9239a842c9e4d9a4238
  ```

</details>

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments.apps coffee
kubectl delete service coffee
kubectl delete gateways cafe-gateway
kubectl delete httproutes coffee
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [Coffee Backend Application](coffee.yaml)
- [Coffee Gateway](gateway.yaml)
- [Coffee HTTPRoute](coffee-httpRoute.yaml)

Previous: [SDE NGINX Gateway Fabric](../README.md)

Next: [Interactive Lab](../lab/README.md)

---

## End of lab
