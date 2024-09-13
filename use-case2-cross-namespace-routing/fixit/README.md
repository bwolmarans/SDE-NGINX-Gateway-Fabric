# Fix the Sock store

This lab is intended to be completed by individuals or small groups during the Up skilling sessions but it maybe completed at anytime.

## Introduction

In this Fix it lab try not to look to closely at the sample code and just copy and run it.  This should result in a broken coats store.  Your challenge is to find and fix the problem and get the coats store back in business.

## Fix it lab

Copy and paste the following code snippet to deploy the **coats** store.

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
  name: coats
  namespace: retail
spec:
  replicas: 1
  selector:
    matchLabels:
      app: coats
  template:
    metadata:
      labels:
        app: coats
    spec:
      containers:
      - name: coats
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: coats
  namespace: retail
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: coats
---
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
    hostname: coats.lab.f5npi.net
---
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
        value: /coats
    backendRefs:
    - name: coats
      namespace: retail
      port: 80
---
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
    namespace: retail
EOF
```

The customers have noticed that the coats store is working as expected but nobody can reach the coats store.  The Infrastructure administrators, Cluster administrators and Application developers all claim to have deployed and configured their objects with no errors and cannot explain at this time why things are not working as expected.  They have called you in to help resolve this issue.

You can start with a quick check for the pods and services and then move on from there...

```bash
kubectl -n retail get all -o wide
```

Next test the basic functionality for **/coats**

```bash
curl http://coats.lab.f5npi.net/coats
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ curl http://coats.lab.f5npi.net/coats
  Server address: 10.244.191.146:8080
  Server name: coats-675d846bb4-kk9j8
  Date: 15/Jul/2024:21:13:20 +0000
  URI: /coats
  Request ID: 8dcb8cdd5cf1dd01685153d363feb1b7
  ```

</details>

Now consider the validation steps we discussed in the lecture and try to apply those now.

<details>
  <summary><b>Solution</b></summary>

  The wrong **namespace** was configured in the **referenceGrant** object.  In the **from**
  section, the **HTTPRoute** references the **retail** namespace when it is suppose to reference
  **default**.

  [Click here](coats-referenceGrant.solution.yaml) to see a solution for the **referenceGrant**
  configuration.

</details>

## Clean up

Run the following commands to remove the objects created in this lab.

```bash
kubectl -n retail delete deployments.apps coats
kubectl -n retail delete services coats
kubectl delete httproutes retail-httproute
kubectl delete gateways retail-gateway
kubectl -n retail delete referencegrants access-to-retail-services
kubectl delete namespace retail
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [Coats application and service](coats.yaml)
- [Retail Gateway](coats-gateway.yaml)
- [HTTPRoute for coats](coats-httpRoute.yaml)

## Solution

- [Reference Grant for coats](coats-referenceGrant.yaml)

Return: [SDE NGINX Gateway Fabric](../README.md)

Previous: [Interactive Lab](../lab/README.md)

Next: [Modify Request Headers Use Case](../../use-case3-mod-req-headers/README.md)

---

## End of lab
