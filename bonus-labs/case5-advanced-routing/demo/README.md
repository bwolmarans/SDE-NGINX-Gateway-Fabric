# Advanced Routing Demo

## Introduction

In this demo we'll imagine that the web app has two pods: One with static content and one that handles POSTed incoming data from web clients.
We'll use a single gateway with a HTTPRoute object that performs HTTP routing based on **Request Method**. This requires two pods and services: One for handling ```POST``` requests and one for handling ```GET``` requests.


## Prepare the lab environment applications that will provide the app servers

These 2 app pods and services respond with a  simple message that indicates their identity.


> If you prefer to manually create and deploy the **post-n-get** application, click [here](post-app.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: post-config
data:
  default.conf: |
    server {
      listen 8080;
      return 200 'Hello from the POST server!\r\n';
      add_header Content-Type text/plain;
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: get-config
data:
  default.conf: |
    server {
      listen 8080;
      return 200 'Hello from the GET server!\r\n';
      add_header Content-Type text/plain;
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: post
spec:
  replicas: 1
  selector:
    matchLabels:
      app: post
  template:
    metadata:
      labels:
        app: post
    spec:
      volumes:
        - name: post-config-volume
          configMap:
            name: post-config
            items:
            - key: default.conf
              path: default.conf
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: post-config-volume
          mountPath: /etc/nginx/conf.d
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: get
spec:
  replicas: 1
  selector:
    matchLabels:
      app: get
  template:
    metadata:
      labels:
        app: get
    spec:
      volumes:
        - name: get-config-volume
          configMap:
            name: get-config
            items:
            - key: default.conf
              path: default.conf
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: get-config-volume
          mountPath: /etc/nginx/conf.d
---      
apiVersion: v1
kind: Service
metadata:
  name: post
spec:
  selector:
    app: post
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
---
apiVersion: v1
kind: Service
metadata:
  name: get
spec:
  selector:
    app: get
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
EOF
```


<details><summary>App creation example response</summary>

```
root@bastion:/# k apply -f post-app.yaml 
configmap/post-config created
configmap/get-config created
deployment.apps/post created
deployment.apps/get created
service/post created
service/get created
```

</details>


Now check your new pods and services.


```bash
kubectl get pod,svc -owide
```

You should see 2 pods and 2 services


## Configure the Gateway

There are 2 **services** that we must expose to the Internet, **get** and **post**. Both of these are exposed on a single endpoint, so we'll need a single kubernetes **gateway** object.

Copy and paste the code block below to create the Gateway.

> If you prefer to manually create and deploy the Gateway, click [here](post-gateway.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: post-n-get
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
EOF
```

<details><summary>Gateway creation example response</summary>

```
root@bastion:/# k create -f post-gateway.yaml 
gateway.gateway.networking.k8s.io/post-n-get created
```

</details>


## Configure the HTTPRoute

We'll need an [HTTPRoute](https://gateway-api.sigs.k8s.io/api-types/httproute/) with 2 [HTTPRouteRules](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPRouteRule). One rule will catch [GETs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET) and the other will catch [POSTs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/POST).

Copy and paste the code block below to create the HTTPRoute.

> If you prefer to manually create and deploy the Gateway, click [here](post-routes.yaml) for the
> YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: post-n-get
spec:
  parentRefs:
  - name: post-n-get
    sectionName: http
  hostnames:
  - coffee.lab.f5npi.net
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
      method: "POST"
    backendRefs:
    - name: post
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
      method: "GET"
    backendRefs:
    - name: get
      port: 80
EOF
```


Review the HTTPRoute.

```bash
kubectl get httproutes 
```


# Testing

## Current httpRoutes resources configured

Ensure the backend reference and gateway reference are resolved from the HTTPRoute object.


```bash
f5admin@bastion:~$ k describe httproutes
```

Note the ```Rule``` section and how each contains ```Matches``` and ```Backend Refs```.

Note the ```Status``` section, it should indicate that the references are resolved because the backend apps "get" and "post" both exist.


## Test a GET

Issue a curl command that sends a GET request:

```
curl -v http://coffee.lab.f5npi.net/
```

You should see a response from the GET server.

<details><summary>Example response</summary>

```
root@bastion:/# curl -v http://coffee.lab.f5npi.net/
*   Trying 10.1.10.100:80...
* Connected to coffee.lab.f5npi.net (10.1.10.100) port 80 (#0)
> GET / HTTP/1.1
> Host: coffee.lab.f5npi.net
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.27.0
< Date: Fri, 12 Jul 2024 15:43:44 GMT
< Content-Type: application/octet-stream
< Content-Length: 28
< Connection: keep-alive
< 
Hello from the GET server!
```

</details>

## Test a POST

Issue a curl command that sends a POST request:

```
curl -v -d 'post data' http://coffee.lab.f5npi.net/
```

<details><summary>Example response</summary>

```
root@bastion:/# curl -v -d 'post data'http://coffee.lab.f5npi.net/
curl: no URL specified!
curl: try 'curl --help' or 'curl --manual' for more information
root@bastion:/# curl -v -d 'post data' http://coffee.lab.f5npi.net/
*   Trying 10.1.10.100:80...
* Connected to coffee.lab.f5npi.net (10.1.10.100) port 80 (#0)
> POST / HTTP/1.1
> Host: coffee.lab.f5npi.net
> User-Agent: curl/7.81.0
> Accept: */*
> Content-Length: 9
> Content-Type: application/x-www-form-urlencoded
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.27.0
< Date: Fri, 12 Jul 2024 15:46:31 GMT
< Content-Type: application/octet-stream
< Content-Length: 29
< Connection: keep-alive
< 
Hello from the POST server!
```

</details>


## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments get post
kubectl delete services get post
kubectl delete gateway post-n-get
kubectl delete httproutes post-n-get
kubectl delete configmaps post-config get-config
```

>**Note**: If you would prefer to manually create and deploy the configurations you can find the source YAML files here:

- [post-app.yaml](post-app.yaml)
- [post-gateway.yaml](post-gateway.yaml)
- [post-routes.yaml](post-routes.yaml)

Previous: [SDE NGINX Gateway Fabric](../README.md)

Next: [Interactive Lab](../lab/README.md)

---

## End of lab
