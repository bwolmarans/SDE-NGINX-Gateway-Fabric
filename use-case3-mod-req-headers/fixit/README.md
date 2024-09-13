# FixIti: RequestHeaderModifier Not Removing Header Lab

## Introduction

WhatIgot Inc. recently had a sale that provided a 50% discount for customers that presented the header **X-Apply-Discount: true**.  However the sale is concluded but the website still appears to be setting this header value for customers browsing the WhatIgot online shop.  While the application team digs through the website and finds the errant code they have asked you to remove the header value prior to any client making request to /cart.  They have been able to enable compression for all clients.  

In this FixIt lab, try not to look too closely at the sample code, just copy and run it the yaml file to create the scenario.  This should result in unexpected behavior.  Your challenge is to find and fix the problem and resolve
the issue.

## Fix it lab

Deploy the lab including the backend, gateway, and HTTPRoute by running the command below.

> If you prefer to manually create and deploy this application, click [here](shop-whatigot-fixit.yaml) for the YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shop
  template:
    metadata:
      labels:
        app: shop
    spec:
      containers:
      - name: shop
        image: nginx
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: shop-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: shop-config
data:
  nginx.conf: |-
    user  nginx;
    worker_processes  1;

    pid        /var/run/nginx.pid;

    load_module /usr/lib/nginx/modules/ngx_http_js_module.so;

    events {}

    http {
        default_type text/plain;

        js_import /etc/nginx/headers.js;
        js_set \$headers headers.getRequestHeaders;

        server {
            listen 8080;
            return 200 "\$headers";
        }
    }
  headers.js: |-
    function getRequestHeaders(r) {
        let s = "Headers:\n";
        for (let h in r.headersIn) {
        s += \`  header '\${h}' is '\${r.headersIn[h]}'\n\`;
        }
        return s;
    }
    export default {getRequestHeaders};
---
apiVersion: v1
kind: Service
metadata:
  name: shop
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: shop
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shop-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-httproute
spec:
  parentRefs:
  - name: shop-gateway
    sectionName: http
  hostnames:
  - "whatigot.lab.f5npi.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /cart 
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: Accept-Encoding
          value: compress
        remove:
        - X-Apply-Discounts
    backendRefs:
    - name: shop
      port: 80
EOF
```

You arrive to work and find the NGINX Gateway Fabric that supports **whatigot** is not working correctly. The Kubernetes administrators believe they have deployed a configuration to remove the incoming requests header `X-Apply-Discount` but that does not appear to be the case.  Now they are looking to you to figure it out.

You know that using `curl` you can test the functionality by simply setting the **X-Apply-Discount** header with the value **true**. Below is the `curl` command you use to start testing.

```bash
curl -v http://whatigot.lab.f5npi.net/cart -H 'X-Apply-Discount: true'
```

<details>
  <summary><h3>Example output</h3></summary>

  ```bash
f5admin@bastion:~/labs/header-modification$ curl -v http://whatigot.lab.f5npi.net/cart -H 'X-Apply-Discount: true'
*   Trying 10.1.10.100:80...
* Connected to whatigot.lab.f5npi.net (10.1.10.100) port 80 (#0)
> GET /cart HTTP/1.1
> Host: whatigot.lab.f5npi.net
> User-Agent: curl/7.81.0
> Accept: */*
> X-Apply-Discount: true
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.27.0
< Date: Mon, 15 Jul 2024 09:34:03 GMT
< Content-Type: text/plain
< Content-Length: 274
< Connection: keep-alive
< 
Headers:
  header 'Accept-Encoding' is 'compress'
  header 'Host' is 'whatigot.lab.f5npi.net'
  header 'X-Forwarded-For' is '10.1.10.11'
  header 'Connection' is 'close'
  header 'User-Agent' is 'curl/7.81.0'
  header 'Accept' is '*/*'
  header 'X-Apply-Discount' is 'true'
* Connection #0 to host whatigot.lab.f5npi.net left intact
  ```

</details>

## Second challenge

The web team would like the web server to respond with the content type **Content-Type: application/json** in addition to any content type the client sends by default. In addition they want to make sure all clients are getting the latest content and would like to push a no-cache directive **Cache-Control: no-cache** as the only cache-control directive.

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployment shop
kubectl delete services shop
kubectl delete configmap shop-config
kubectl delete gateways shop-gateway 
kubectl delete httproutes shop-httproute
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [What I Got Backend Application](shop-whatigot.yaml)
- [What I Got Gateway](shop-gateway.yaml)
- [What I Got HTTPRoute](shop-whatigot-httpRoute.yaml)

## Solution 

Here is the solution for reference.

- [What I got fixed](shop-whatigot-fixed.yaml)

Return: [SDE NGINX Gateway Fabric](../README.md)

Previous: [Interactive Lab](../lab/README.md)

Next: [HTTPS Termination Use Case](../../use-case4-https-termination/README.md)

---

End of lab

---
