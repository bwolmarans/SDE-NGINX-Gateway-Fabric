# Hats Demo

This lab is intended for the instruction to demonstrate this use case using the instructions from
this page. Users can either watch or following along using their own live environment.

## Introduction

In this demo we are going to deploy the [Hats application and service](hats.yaml) to a new
namespace called **retail**. The gateway object and **HTTPRoute** will be deployed to the
**default** namespace but our application developers are working in the **retail** namespace in
this lab.  As a result we need to add a **ReferenceGrant** object that will allow traffic between
these two namespaces.  This is a task that might be completed by the **Applications Developer**.

## Demo Lab

Copy and paste the following code snippet to deploy the hats application and service.

> If you prefer to manually create and deploy this application, click [here](hats.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: retail
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hats
  namespace: retail
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hats
  template:
    metadata:
      labels:
        app: hats
    spec:
      containers:
      - name: hats
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hats
  namespace: retail
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: hats
EOF
```

Now check your new pod and service in the **retail** namespace.

>**Question**: What namespace were the PODS and Services created in?

<details>
  <summary><b>Answer</b></summary>

They are created in the **retail** kubernetes namespace.

</details>

```bash
kubectl -n retail get pod,svc -owide
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ kubectl -n hats get pod,svc -owide

  NAME                        READY   STATUS    RESTARTS   AGE     IP               NODE                    NOMINATED NODE   READINESS GATES
  pod/hats-76dd45b4bb-5vl2q   1/1     Running   0          4m42s   10.244.191.144   w3-mgmt.lab.f5npi.net   <none>           <none>

  NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
  service/hats   ClusterIP   10.103.93.141   <none>        80/TCP    4m42s   app=hats
  ```

</details>

Then we will create an NGINX Gateway Fabric gateway object that will enable traffic
into the retail namespace based on the following criteria.

| Property      | Values                 |
| ------------- | ---------------------- |
| port          | `80`                   |
| protocol      | `HTTP`                 |
| hostname      | `hats.lab.f5npi.net`   |

This could be the responsibility of the **Cluster Administrator**.

Copy and paste the following code snippet to deploy the coffee gateway.

> If you prefer to manually create this gateway, click [here](retail-gateway.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: retail-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: hats.lab.f5npi.net
EOF
```

>**Question**: What namespace was the gateway created in?

<details>
  <summary><b>Answer</b></summary>

It is created in the **default** kubernetes namespace.

</details>

Check the new retail-gateway health.

```bash
kubectl get gateways retail-gateway
kubectl describe gateways retail-gateway
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ kubectl get gateways retail-gateway
  NAME             CLASS   ADDRESS       PROGRAMMED   AGE
  retail-gateway   nginx   10.1.10.100   True         7s

  f5admin@bastion:~$ kubectl describe gateways retail-gateway
  Name:         retail-gateway
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         Gateway
  Metadata:
    Creation Timestamp:  2024-07-15T18:42:04Z
    Generation:          1
    Resource Version:    135144
    UID:                 74a1ac48-8c47-490e-aba1-ae26342ecb67
  Spec:
    Gateway Class Name:  nginx
    Listeners:
      Allowed Routes:
        Namespaces:
          From:  Same
      Hostname:  hats.lab.f5npi.net
      Name:      http
      Port:      80
      Protocol:  HTTP
  Status:
    Addresses:
      Type:   IPAddress
      Value:  10.1.10.100
    Conditions:
      Last Transition Time:  2024-07-15T18:42:04Z
      Message:               Gateway is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-15T18:42:04Z
      Message:               Gateway is programmed
      Observed Generation:   1
      Reason:                Programmed
      Status:                True
      Type:                  Programmed
    Listeners:
      Attached Routes:  0
      Conditions:
        Last Transition Time:  2024-07-15T18:42:04Z
        Message:               Listener is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-15T18:42:04Z
        Message:               Listener is programmed
        Observed Generation:   1
        Reason:                Programmed
        Status:                True
        Type:                  Programmed
        Last Transition Time:  2024-07-15T18:42:04Z
        Message:               All references are resolved
        Observed Generation:   1
        Reason:                ResolvedRefs
        Status:                True
        Type:                  ResolvedRefs
        Last Transition Time:  2024-07-15T18:42:04Z
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

Next we will create the [HTTPRoute](coffee-httpRoute.yaml) that will allow traffic inbound to our
**hats** application and leverage the NGINX Gateway Fabric gateway object.  This object will also
define the path uri of **/hats** and the endpoint of the backend service.  This could be the
responsibility of the **Application Developer**.

Copy and paste the following code snippet to deploy the HTTPRoute for the hats application.

> If you prefer to manually create this **HTTPRoute**, click [here](hats-httpRoute.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: retail-httproute
spec:
  parentRefs:
  - name: retail-gateway
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /hats
    backendRefs:
    - name: hats
      namespace: retail
      port: 80
EOF
```

Check the new httpRoute health.

```bash
kubectl get httproutes retail-httproute
kubectl describe httproutes retail-httproute
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ kubectl get httproutes retail-httproute
  NAME               HOSTNAMES   AGE
  retail-httproute               23s

  f5admin@bastion:~$ kubectl describe httproutes retail-httproute
  Name:         retail-httproute
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         HTTPRoute
  Metadata:
    Creation Timestamp:  2024-07-15T19:01:31Z
    Generation:          1
    Resource Version:    138188
    UID:                 c54f7dec-f130-421a-bf76-407e763ad901
  Spec:
    Parent Refs:
      Group:         gateway.networking.k8s.io
      Kind:          Gateway
      Name:          retail-gateway
      Section Name:  http
    Rules:
      Backend Refs:
        Group:
        Kind:       Service
        Name:       hats
        Namespace:  retail
        Port:       80
        Weight:     1
      Matches:
        Path:
          Type:   PathPrefix
          Value:  /hats
  Status:
    Parents:
      Conditions:
        Last Transition Time:  2024-07-15T19:01:31Z
        Message:               The route is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-15T19:01:31Z
        Message:               Backend ref to Service retail/hats not permitted by any ReferenceGrant
        Observed Generation:   1
        Reason:                RefNotPermitted
        Status:                False
        Type:                  ResolvedRefs
      Controller Name:         gateway.nginx.org/nginx-gateway-controller
      Parent Ref:
        Group:         gateway.networking.k8s.io
        Kind:          Gateway
        Name:          retail-gateway
        Namespace:     default
        Section Name:  http
  Events:              <none>
  ```

</details>

## Test application

Now test your newly exposed application using the **NGINX Gateway Fabric HTTPRoute** we just
deployed that is linked to the **retail-gateway** object.  However, this time it fails... why?
Because we do not have a referenceGrant configured to allow this traffic.

```bash
curl hats.lab.f5npi.net/hats
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ curl hats.lab.f5npi.net/hats
  <html>
  <head><title>500 Internal Server Error</title></head>
  <body>
  <center><h1>500 Internal Server Error</h1></center>
  <hr><center>nginx/1.27.0</center>
  </body>
  </html>
  ```

</details>

Now we are going to create our hats **ReferenceGrant** allowing routes between the **default** and
**retail** namespaces.

> If you prefer to manually create this **referenceGrant**, click [here](hats-referenceGrant.yaml)
> for the YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: access-to-retail-services
  namespace: retail
spec:
  to:
  - group: ""
    kind: Service
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: default
EOF
```

>**Question**: What namespace was the **referenceGrant** created in?

<details>
  <summary><b>Answer</b></summary>

  It is created in the **retail** namespace. This is the location of the **hats** application.

</details>

Check the new referenceGrant health.

```bash
kubectl get -n retail referencegrants access-to-retail-services
kubectl describe -n retail referencegrants access-to-retail-services
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ kubectl get -n retail referencegrants access-to-retail-services
  NAME                        AGE
  access-to-retail-services   2s

  f5admin@bastion:~$ kubectl describe -n retail referencegrants access-to-retail-services
  Name:         access-to-retail-services
  Namespace:    retail
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1beta1
  Kind:         ReferenceGrant
  Metadata:
    Creation Timestamp:  2024-07-15T19:23:43Z
    Generation:          1
    Resource Version:    141662
    UID:                 49b4200d-904b-44b4-b711-c12f0168d1ae
  Spec:
    From:
      Group:      gateway.networking.k8s.io
      Kind:       HTTPRoute
      Namespace:  default
    To:
      Group:
      Kind:   Service
  Events:     <none>
  ```

</details>

## Test application (part 2)

Now we can test our application again and hopefully this time it works as expected.

```bash
curl hats.lab.f5npi.net/hats
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ curl hats.lab.f5npi.net/hats
  Server address: 10.244.191.144:8080
  Server name: hats-76dd45b4bb-5vl2q
  Date: 15/Jul/2024:19:26:33 +0000
  URI: /hats
  Request ID: f0b605a59b01de94cf0600ffb2e7de74
  ```

</details>

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl -n retail delete deployments.apps hats
kubectl -n retail delete service hats
kubectl delete namespace retail
kubectl delete gateways retail-gateway
kubectl delete httproutes retail-httproute
kubectl delete -n retail referencegrants access-to-retail-services
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [Hats application and service](hats.yaml)
- [Retail Gateway](retail-gateway.yaml)
- [HTTPRoute for Hats](hats-httpRoute.yaml)
- [Reference Grant for Hats](hats-referenceGrant.yaml)

Previous: [SDE NGINX Gateway Fabric](../README.md)

Next: [Interactive Lab](../lab/README.md)

---

## End of lab
