# Shoes lab

This lab is intended to be completed by individuals or small groups during the up-skilling sessions
but it maybe completed at anytime.

## Introduction

In this demo we are going to deploy the Shoes application and service to a new
namespace called **footwear**. The gateway object and **HTTPRoute** will be deployed to the
**default** namespace but our application developers are working in the **footwear** namespace in
this lab.  As a result we need to add a **ReferenceGrant** object that will allow traffic between
these two namespaces.  This is a task that might be completed by the **Applications Developer**.

## Interactive Labs

Copy and paste the following code snippet to deploy the coffee application and service.

> If you prefer to manually create and deploy this application, click [here](shoes.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: footwear
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoes
  namespace: footwear
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shoes
  template:
    metadata:
      labels:
        app: shoes
    spec:
      containers:
      - name: shoes
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: shoes
  namespace: footwear
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: shoes
EOF
```

Now check your new pod and service

>**Question**: What namespace were the PODS and Services created in?

<details>
  <summary><b>Answer</b></summary>

They are created in the **footwear** kubernetes namespace.

</details>

```bash
kubectl -n footwear get pod,svc -owide
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ kubectl -n footwear get pod,svc -owide
  NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE                    NOMINATED NODE   READINESS GATES
  pod/shoes-5df7b98484-mn2xh   1/1     Running   0          13s   10.244.191.145   w3-mgmt.lab.f5npi.net   <none>           <none>

  NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
  service/shoes   ClusterIP   10.111.163.201   <none>        80/TCP    13s   app=shoes
  ```

</details>

Then we will create an NGINX Gateway Fabric gateway object that will enable traffic into the
**default** namespace based on the following criteria.

| Property      | Values                 |
| ------------- | ---------------------- |
| port          | `80`                   |
| protocol      | `HTTP`                 |
| hostname      | `shoes.lab.f5npi.net`  |

This could be the responsibility of the **Cluster Administrator**.

Copy and paste the following code snippet to deploy the shoes gateway.

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
    hostname: shoes.lab.f5npi.net
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
  retail-gateway   nginx   10.1.10.100   True         5s

  f5admin@bastion:~$ kubectl describe gateways retail-gateway
  Name:         retail-gateway
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         Gateway
  Metadata:
    Creation Timestamp:  2024-07-15T19:46:25Z
    Generation:          1
    Resource Version:    145271
    UID:                 16ab8079-71a9-4015-b389-c97114652d22
  Spec:
    Gateway Class Name:  nginx
    Listeners:
      Allowed Routes:
        Namespaces:
          From:  Same
      Hostname:  shoes.lab.f5npi.net
      Name:      http
      Port:      80
      Protocol:  HTTP
  Status:
    Addresses:
      Type:   IPAddress
      Value:  10.1.10.100
    Conditions:
      Last Transition Time:  2024-07-15T19:46:25Z
      Message:               Gateway is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-15T19:46:25Z
      Message:               Gateway is programmed
      Observed Generation:   1
      Reason:                Programmed
      Status:                True
      Type:                  Programmed
    Listeners:
      Attached Routes:  0
      Conditions:
        Last Transition Time:  2024-07-15T19:46:25Z
        Message:               Listener is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-15T19:46:25Z
        Message:               Listener is programmed
        Observed Generation:   1
        Reason:                Programmed
        Status:                True
        Type:                  Programmed
        Last Transition Time:  2024-07-15T19:46:25Z
        Message:               All references are resolved
        Observed Generation:   1
        Reason:                ResolvedRefs
        Status:                True
        Type:                  ResolvedRefs
        Last Transition Time:  2024-07-15T19:46:25Z
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

Next we will create the **HTTPRoute** that will allow traffic inbound to our **hats** application
and leverage the NGINX Gateway Fabric gateway object.  This object will also
define the path uri of **/shoes** and the endpoint of the backend service.  This could be the
responsibility of vhe **Application Developer**.

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
        value: /shoes
    backendRefs:
    - name: shoes
      namespace: footwear
      port: 80
EOF
```

Next we will create the **referenceGrant**.  This is the object that explicitly allows traffic to
route to the **footwear** namespace.

In an effort to make this just a little more challenging this code snippet will create file called
**shoes-referenceGrant.yaml** in your current working directory. You will need to update the values
that are surrounded in **##** symbols.

> **Note**: This code snippet will create a file that you must **edit** manually and then deploy with `kubectl create -f shoes-referenceGrant.yaml`.

```bash
echo "apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: access-to-footwear-services
  namespace: ##What Namespace##
spec:
  to:
  - group: \"\"
    kind: Service
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: ##What Namespace##" > shoes-referenceGrant.yaml
```

Once you have fixed the YAML configuration file create the object using the command below

```bash
kubectl create -f shoes-referenceGrant.yaml
```

Now you can check on the status of your new tea httpRoute rule.

```bash
kubectl -n footwear get referenceGrant access-to-footwear-services
kubectl -n footwear describe referenceGrant access-to-footwear-services
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$` kubectl -n footwear get referenceGrant access-to-footwear-services
  NAME                          AGE
  access-to-footwear-services   28s
  f5admin@bastion:~$ kubectl -n footwear describe referenceGrant access-to-footwear-services
  Name:         access-to-footwear-services
  Namespace:    footwear
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1beta1
  Kind:         ReferenceGrant
  Metadata:
    Creation Timestamp:  2024-07-15T20:14:27Z
    Generation:          1
    Resource Version:    149655
    UID:                 fc06b602-6910-468b-97f7-be1951823236
  Spec:
    From:
      Group:      gateway.networking.k8s.io
      Kind:       HTTPRoute
      Namespace:  default
    To:
      Group:
      Kind:   Service
  Events:     <none>`
  ```

</details>

## Test Application

Now test your newly exposed application using the **NGINX Gateway Fabric HTTPRoute** we just deployed and that is linked the **cafe-gateway** object.

```bash
curl shoes.lab.f5npi.net/shoes
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ curl shoes.lab.f5npi.net/shoes
  Server address: 10.244.191.145:8080
  Server name: shoes-5df7b98484-mn2xh
  Date: 15/Jul/2024:20:23:24 +0000
  URI: /shoes
  Request ID: ca2ec92816fb536f9063a353124dba93
  ```

</details>

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl -n footwear delete deployments.apps shoes
kubectl -n footwear delete service shoes
kubectl delete gateways retail-gateway
kubectl delete httproutes retail-httproute
kubectl delete -n footwear referencegrants access-to-footwear-services
kubectl delete ns footwear
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [Shoes application and service](shoes.yaml)
- [Retail Gateway](retail-gateway.yaml)
- [HTTPRoute for shoes](shoes-httpRoute.yaml)

## Solution Reference Grant

- [Reference Grant for shoes](shoes-referenceGrant.yaml)

Previous: [Demo](../demo/README.md)

Next: [Fix It Lab](../fixit/README.md)

---

## End of lab
