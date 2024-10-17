# Fix the Sock Store

This lab is intended to be completed by individuals or small groups during the Up skilling sessions
but it maybe completed at anytime.

## Introduction

In this FixIt, try not to look too closely at the sample code and just copy and run it.  This
should result in a broken sock store.  Your challenge is to find and fix the problem and get the
sock store back in business.

## Fix it lab

Copy and paste the following code snippet to deploy the **socks and coffee** store.

```bash
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: socks
spec:
  replicas: 1
  selector:
    matchLabels:
      app: socks
  template:
    metadata:
      labels:
        app: socks
    spec:
      containers:
      - name: socks
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: socks
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: socks
---
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
---
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
    hostname: "*.lab.f5npi.net"
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: coffee-and-socks
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
      port: 40
    - path:
        type: PathPrefix
        value: /socks
    backendRefs:
    - name: socks
      port: 80
EOF
```

The customers have noticed that the coffee and sock store are not working as expected. It seems like one service is unavailable and the other seems to be returning the wrong content. The Infrastructure administrators, Cluster administrators and Application developers all claim to have deployed and configured their objects with no errors and cannot explain at this time why things are not working as expected.  They have called you in to help resolve this issue.

You can start with a quick check for the pods and services and then move on from there...

```bash
kubectl get all -owide
```

Next test the basic functionality for /coffee and /socks

```bash
curl http://coffee.lab.f5npi.net/coffee
```
```bash
curl http://socks.lab.f5npi.net/socks
```

Now consider the validation steps we discussed in the lecture and try to apply those now. **Fix it**.

## Clean up

Run the following commands to remove the objects created in this lab.

```bash
kubectl delete deployments.apps coffee socks
kubectl delete services coffee socks
kubectl delete httproutes coffee-and-socks
kubectl delete gateways cafe-gateway
```

<details>
  <summary><h3><b>Solution</b><h3></summary>

  [Click here](httproute-combined-solution.yaml) to see a solution for a **HTTPRoute** configuration.
</details>

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [Coffee and Socks Backend Application](coffee-and-socks.yaml)
- [Coffee and Socks Gateway](gateway.yaml)
- [Coffee and Socks HTTP Route](httproute-combined.yaml)

Return: [SDE NGINX Gateway Fabric](../README.md)

Previous: [Interactive Lab](../lab/README.md)

Next: [Cross Namespace Routing Use Case](../../use-case2-cross-namespace-routing/README.md)

---

## End of lab
