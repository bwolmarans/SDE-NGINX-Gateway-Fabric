# Tea Lab

This lab is intended to be completed by individuals or small groups during the Up skilling sessions but it maybe completed at anytime.

## Introduction

In this demo we are going to deploy the tea application and service to the default namespace.  This step could be the responsibility of the **Applications Developer**.

## Interactive lab

Copy and paste the following code snippet to deploy the coffee application and service.

> If you want to see the yaml file for this application, click [here](tea.yaml).

```bash
kubectl create -f - <<EOF
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
  name: tea
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

Now check your new pod and service

```bash
kubectl get pod,svc -owide
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ kubectl get pod,svc -owide
  NAME                      READY   STATUS    RESTARTS   AGE   IP              NODE                    NOMINATED NODE   READINESS GATES
  pod/tea-9d8868bb4-fqqkd   1/1     Running   0          24m   10.244.67.147   w1-mgmt.lab.f5npi.net   <none>           <none>

  NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE    SELECTOR
  service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   5d6h   <none>
  service/tea          ClusterIP   10.98.25.167   <none>        80/TCP    24m    app=tea
  ```

</details>

Then we will create an **NGINX Gateway Fabric Gateway** that will enable traffic into the default
namespace based on the following criteria.

| Property      | Values                 |
| ------------- | ---------------------- |
| port          | `80`                   |
| protocol      | `HTTP`                 |
| hostname      | `tea.lab.f5npi.net`    |

This could be the responsibility of the **cluster administrator**.

Copy and paste the following code snippet to deploy the tea gateway.

> If you want to see the yaml file for this gateway, click [here](gateway.yaml).

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
    hostname: tea.lab.f5npi.net
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
  f5admin@bastion:~$ kubectl get gateways cafe-gateway
  NAME           CLASS   ADDRESS       PROGRAMMED   AGE
  cafe-gateway   nginx   10.1.10.100   True         33s

  f5admin@bastion:~$ kubectl describe gateways cafe-gateway
  Name:         cafe-gateway
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         Gateway
  Metadata:
    Creation Timestamp:  2024-07-11T06:43:21Z
    Generation:          1
    Resource Version:    161881
    UID:                 2de17c8f-a9fa-49a7-9c15-6f3a4db06509
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
      Last Transition Time:  2024-07-11T06:43:22Z
      Message:               Gateway is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-11T06:43:22Z
      Message:               Gateway is programmed
      Observed Generation:   1
      Reason:                Programmed
      Status:                True
      Type:                  Programmed
    Listeners:
      Attached Routes:  0
      Conditions:
        Last Transition Time:  2024-07-11T06:43:22Z
        Message:               Listener is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-11T06:43:22Z
        Message:               Listener is programmed
        Observed Generation:   1
        Reason:                Programmed
        Status:                True
        Type:                  Programmed
        Last Transition Time:  2024-07-11T06:43:22Z
        Message:               All references are resolved
        Observed Generation:   1
        Reason:                ResolvedRefs
        Status:                True
        Type:                  ResolvedRefs
        Last Transition Time:  2024-07-11T06:43:22Z
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

Next we will create the **HTTPRoute** that will allow traffic inbound to our coffee application and leverage the NGINX Gateway Fabric gateway object.  This object will also define the path uri of **/tea** and backend service (POD) to route traffic to.  This could be the responsibility of the **Application Developer**.


>**Note**: This code snippet will create a file that you must update and then deploy with `kubectl create -f tea-httpRoute.yaml`.
>**Note**: To make this lab just slightly challenging the following code snippet will create a file called **tea-httpRoute.yaml** but it will not work until you update the values surrounded in **##** symbols.  Refer back to how we created the **coffee* route in the demo for reference. 

```bash
echo "apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ##NAME##
spec:
  parentRefs:
  - name: ##WHAT Gateway##
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /tea
    backendRefs:
    - name: ##WHAT SERVICE##
      port: ##WHAT PORT## " >tea-httpRoute.yaml
```

Once you have fixed the YAML configuration file, create the object using the command below.

```bash
kubectl create -f tea-httpRoute.yaml
```

Now you can check on the status of your new tea httpRoute rule.

```bash
kubectl get httproutes tea
```
```bash
kubectl describe httproutes tea
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ kubectl describe httproutes tea
  Name:         tea
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         HTTPRoute
  Metadata:
    Creation Timestamp:  2024-07-11T06:38:14Z
    Generation:          1
    Resource Version:    161070
    UID:                 62fd7e19-a8f3-4c32-97ec-5be39b4ff372
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
        Name:    tea
        Port:    80
        Weight:  1
      Matches:
        Path:
          Type:   PathPrefix
          Value:  /tea
  Status:
    Parents:
      Conditions:
        Last Transition Time:  2024-07-11T06:38:14Z
        Message:               The route is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-11T06:38:14Z
        Message:               All references are resolved
        Observed Generation:   1
        Reason:                ResolvedRefs
        Status:                True
        Type:                  ResolvedRefs
        Last Transition Time:  2024-07-11T06:38:14Z
        Message:               The condition for this has not been implemented yet: Gateway is ignored
        Observed Generation:   1
        Reason:                TODO
        Status:                True
        Type:                  TODO
      Controller Name:         gateway.nginx.org/nginx-gateway-controller
      Parent Ref:
        Group:         gateway.networking.k8s.io
        Kind:          Gateway
        Name:          cafe-gateway
        Namespace:     default
        Section Name:  http
  Events:              <none>

  f5admin@bastion:~$ kubectl get httproutes tea
  NAME   HOSTNAMES   AGE
  tea                2s
  ```

</details>

## Testing

Now test your newly exposed application using the **NGINX Gateway Fabric HTTPRoute** we just deployed and that is linked the **cafe-gateway** object.

```bash
curl tea.lab.f5npi.net/tea
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/tea
  Server address: 10.244.67.147:8080
  Server name: tea-9d8868bb4-fqqkd
  Date: 11/Jul/2024:06:45:07 +0000
  URI: /tea
  Request ID: d8af59c930f64d3acfacc3cad21ac073
  ```

</details>

<details>
  <summary><h3><b>Solution</b><h3></summary>

  [Click here](tea-httpRoute.yaml) to see a solution for a <b>HTTPRoute</b> configuration.
</details>


## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments.apps tea
kubectl delete service tea
kubectl delete gateways cafe-gateway
kubectl delete httproutes tea
```

---

## Bonus challenge

Can you create another **HTTPRoute** so this use case supports two rules: one for **coffee** and
one for **tea**?  This means you will need to deploy the **coffee application and service** again.
In addition, you will have to update the **Gateway** to support both `coffee.npi.f5net.com` and
`tea.npi.f5net.com`.

Once done you can test your solution using the following two curl commands.

```bash
curl tea.lab.f5npi.net/tea
```
```bash
curl coffee.lab.f5npi.net/coffee
```

But the following commands should return an error.

```bash
curl tea.lab.f5npi.net/coffee
curl coffee.lab.f5npi.net/tea
```

<details>
  <summary><h3<b>Bonus challenge solution</b><h3></summary>

  The links below are the yaml files to configure the coffee application, <b>Gateway</b>, and
  <b>HTTPRoute</b> to enable both the <b>tea</b> and <b>coffee</b> applications

<!-- markdownlint-disable MD007 -->
  - [Coffee application and service](../demo/coffee.yaml)
  - [Coffee and Tea Gateway](bonus/coffee-tea-gateway.yaml)
  - [Coffee HTTPRoute](bonus/coffee-httpRoute.yaml)
  - [Tea HTTPRoute](bonus/tea-httpRoute.yaml)
<!-- markdownlint-enable MD007 -->
</details>


Next, and to prove the relationship of hostnames in Gateways to HTTP Routes, update the Gateway and the Routes to use **cafe.npi.f5net.com**.
You can delete and recreate, or use kubectl apply to modify.

<details>
  <summary><h4>Solution</h4></summary>
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: cafe-gateway
  spec:
    gatewayClassName: nginx
    listeners:
    - name: http-cafe
      port: 80
      protocol: HTTP
      hostname: cafe.lab.f5npi.net
  EOF
  ```
  ```bash  
  k apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: route-coffee
  spec:
    parentRefs:
    - name: cafe-gateway
      sectionName: http-cafe
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
  k apply -f - <<EOF
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: route-tea
  spec:
    parentRefs:
    - name: cafe-gateway
      sectionName: http-cafe
    rules:
    - matches:
      - path:
          type: PathPrefix
          value: /tea
      backendRefs:
      - name: tea
        port: 80
  EOF
  ```
</details>

## Test

```bash
curl cafe.lab.f5npi.net/coffee
```
```bash
curl cafe.lab.f5npi.net/tea
```

  Instead of using the above [Coffee and Tea Gateway](bonus/coffee-tea-gateway.yaml), what would
  happen if you used the example config below where the **hostname** field is missing.

  ```yaml
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
  ```

  <details>
    <summary><b>Answer</b></summary>

  The following commands will work because the configuration allows any hostname by default.
  Notice the hostname and uri differences. For example on `tea.lab.f5npi.net/coffee` where the
  hostname is `tea.lab.f5npi.net` but the uri is `coffee`.

  ```bash
  curl tea.lab.f5npi.net/tea
  curl coffee.lab.f5npi.net/coffee
  curl tea.lab.f5npi.net/coffee
  curl coffee.lab.f5npi.net/tea
  ```
</details>

## Clean up

Be sure to clean up your challenge related objects if you created them.

Here are sample commands that might help you with the challenge cleanup.

```bash
kubectl delete deployments.apps tea coffee
kubectl delete service tea coffee
kubectl delete gateways cafe-gateway
kubectl delete httproutes tea coffee
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [Tea Backend Application](tea.yaml)
- [Tea Gateway](gateway.yaml)
- [Tea HTTPRoute](tea-httpRoute.yaml)

Previous: [Demo](../demo/README.md)

Next: [Fix It Lab](../fixit/README.md)

---

## End of lab
