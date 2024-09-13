# RequestHeaderModifier Demo

This lab is intended for the instruction to demonstrate this use case using the instructions from
this page. Users can either watch or following along using their own live environment.

## Introduction

## Demo

Copy and paste the following code snippet to deploy the **whatigot** application,service and config map.  The **whatigot** application is providing a web server with an echo function that will respond with the values inserted by the NGINX Gateway Fabric **HTTPRoute** `RequestHeaderModifier` filter.

> If you prefer to manually create and deploy this application, click [here](whatigot.yaml) for the YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whatigot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whatigot
  template:
    metadata:
      labels:
        app: whatigot
    spec:
      containers:
      - name: whatigot
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
          name: whatigot-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: whatigot-config
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
  name: whatigot
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: whatigot
EOF
```

Now check your new pod, service, and config map

```bash
kubectl get pod,svc,cm -owide
```

<details>
  <summary><b>Example output</b></summary>

   ```bash
  f5admin@bastion:~/$ kubectl get pod,svc,cm -owide
  NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE                    NOMINATED NODE   READINESS GATES
  pod/whatigot-b899d9bc5-8ptcl   1/1     Running   0          47m   10.244.191.150   w3-mgmt.lab.f5npi.net   <none>           <none>

  NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
  service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   5d19h   <none>
  service/whatigot     ClusterIP   10.109.116.221   <none>        80/TCP    47m     app=whatigot

  NAME                         DATA   AGE
  configmap/kube-root-ca.crt   1      5d19h
  configmap/whatigot-config    2      47m
  ```

</details>

You may also want to review the configmap details for **whatigot-config**.  Keep in mind the configuration map is part of the **whatigot** demo application and not used by the **NGINX Gateway Fabric**. Additional details about the [NGINX javascript module](https://nginx.org/en/docs/http/ngx_http_js_module.html) in use can be found on the [nginx.org](https://nginx.org/) site.

```bash
kubectl describe configmaps whatigot-config
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
f5admin@bastion:~$ kubectl describe configmaps whatigot-config
Name:         whatigot-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
headers.js:
----
function getRequestHeaders(r) {
    let s = "Headers:\n";
    for (let h in r.headersIn) {
    s += `  header '${h}' is '${r.headersIn[h]}'\n`;
    }
    return s;
}
export default {getRequestHeaders};
nginx.conf:
----
user  nginx;
worker_processes  1;

pid        /var/run/nginx.pid;

load_module /usr/lib/nginx/modules/ngx_http_js_module.so;

events {}

http {
    default_type text/plain;

    js_import /etc/nginx/headers.js;
    js_set $headers headers.getRequestHeaders;

    server {
        listen 8080;
        return 200 "$headers";
    }
}

BinaryData
====

Events:  <none>
```

</details>

Then we will create an [NGINX Gateway Fabric gateway](gateway.yaml) object that will enable traffic into the default namespace based on the following criteria.

| Name                   | Value     |
| ---------------------- | -------   |
| **gateway name**       | `gateway` |
| **gatewayClassName**   | `nginx`   |
| Listener **name**      | `http`    |
| Listener **port**      | `80`      |
| Listener **protocol**  | `HTTP`    |

This could be the responsibility of the **Cluster Administrator**.

Copy and paste the following code snippet to deploy the **whatigot** gateway.

> If you prefer to manually create and deploy this gateway, click [here](gateway.yaml) for the YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
EOF
```

Check the new gateway health.

```bash
kubectl get gateways gateway
kubectl describe gateways gateway
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~/$ kubectl get gateways gateway
  NAME      CLASS   ADDRESS       PROGRAMMED   AGE
  gateway   nginx   10.1.10.100   True         49m

  f5admin@bastion:~/$ kubectl describe gateways gateway
  Name:         gateway
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         Gateway
  Metadata:
    Creation Timestamp:  2024-07-11T18:24:56Z
    Generation:          1
    Resource Version:    199873
    UID:                 0987a0f9-b22a-47fb-9fae-d5dcc92c0ddf
  Spec:
    Gateway Class Name:  nginx
    Listeners:
      Allowed Routes:
        Namespaces:
          From:  Same
      Name:      http
      Port:      80
      Protocol:  HTTP
  Status:
    Addresses:
      Type:   IPAddress
      Value:  10.1.10.100
    Conditions:
      Last Transition Time:  2024-07-11T18:25:05Z
      Message:               Gateway is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2024-07-11T18:25:05Z
      Message:               Gateway is programmed
      Observed Generation:   1
      Reason:                Programmed
      Status:                True
      Type:                  Programmed
    Listeners:
      Attached Routes:  1
      Conditions:
        Last Transition Time:  2024-07-11T18:25:05Z
        Message:               Listener is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-11T18:25:05Z
        Message:               Listener is programmed
        Observed Generation:   1
        Reason:                Programmed
        Status:                True
        Type:                  Programmed
        Last Transition Time:  2024-07-11T18:25:05Z
        Message:               All references are resolved
        Observed Generation:   1
        Reason:                ResolvedRefs
        Status:                True
        Type:                  ResolvedRefs
        Last Transition Time:  2024-07-11T18:25:05Z
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

Copy and paste the following code snippet to create the **whatigot** HTTPRoute.

> If you prefer to manually create this HTTPRoute, click [here](whatigot-httpRoute.yaml) for the YAML file.

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: whatigot
spec:
  parentRefs:
  - name: gateway
    sectionName: http
  hostnames:
  - "whatigot.lab.f5npi.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /whatigot
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        set:
        - name: My-favorite-band-today
          value: Stryper
        add:
        - name: My-list-of-bands
          value: Simply-Red
        remove:
        - User-Agent
    backendRefs:
    - name: whatigot
      port: 80
EOF
```

Check the health of the new HTTPRoute.

```bash
kubectl get httproutes whatigot
kubectl describe httproutes whatigot
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~/$ kubectl get httproutes whatigot
  NAME       HOSTNAMES                    AGE
  whatigot   ["whatigot.lab.f5npi.net"]   51m

  f5admin@bastion:~/$ kubectl describe httproutes whatigot
  Name:         whatigot
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  API Version:  gateway.networking.k8s.io/v1
  Kind:         HTTPRoute
  Metadata:
    Creation Timestamp:  2024-07-11T18:25:04Z
    Generation:          1
    Resource Version:    199872
    UID:                 fa62e16f-6ff5-4b54-bad2-95a705500167
  Spec:
    Hostnames:
      whatigot.lab.f5npi.net
    Parent Refs:
      Group:         gateway.networking.k8s.io
      Kind:          Gateway
      Name:          gateway
      Section Name:  http
    Rules:
      Backend Refs:
        Group:
        Kind:    Service
        Name:    whatigot
        Port:    80
        Weight:  1
      Filters:
        Request Header Modifier:
          Add:
            Name:   My-list-of-bands
            Value:  Simply-Red
          Remove:
            User-Agent
          Set:
            Name:   My-favorite-band-today
            Value:  Stryper
        Type:       RequestHeaderModifier
      Matches:
        Path:
          Type:   PathPrefix
          Value:  /whatigot
  Status:
    Parents:
      Conditions:
        Last Transition Time:  2024-07-11T18:25:05Z
        Message:               The route is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-11T18:25:05Z
        Message:               All references are resolved
        Observed Generation:   1
        Reason:                ResolvedRefs
        Status:                True
        Type:                  ResolvedRefs
      Controller Name:         gateway.nginx.org/nginx-gateway-controller
      Parent Ref:
        Group:         gateway.networking.k8s.io
        Kind:          Gateway
        Name:          gateway
        Namespace:     default
        Section Name:  http
  Events:              <none>
  ```

</details>

## Test Application

Now use `curl` to make request to the NGINX Gateway Fabric server and see headers updated per the HTTPRoute filter configuration. We are adding to **My-list-of-bands** and changing the value of **My-favorite-band-today** using the add and set function of the **Request Header Modifier** filter.

```bash
curl -v http://whatigot.lab.f5npi.net/whatigot -H 'My-list-of-bands:Led-Zeppelin,Bee-Gees' -H 'My-favorite-band-today:The-Carpenters'
```

<details>
  <summary><h3>Example output</h3></summary>

  ```bash
  f5admin@bastion:~$ curl -v http://whatigot.lab.f5npi.net/whatigot -H 'My-list-of-bands:Led-Zeppelin,Bee-Gees' -H 'My-favorite-band-today:The-Carpenters'
  *   Trying 10.1.10.100:80...
  * Connected to whatigot.lab.f5npi.net (10.1.10.100) port 80 (#0)
  > GET /whatigot HTTP/1.1
  > Host:whatigot.lab.f5npi.net
  > User-Agent: curl/7.81.0
  > Accept: */*
  > My-list-of-bands:Led-Zeppelin,Bee-Gees
  > My-favorite-band-today:The-Carpenters
  >
  * Mark bundle as not supporting multiuse
  < HTTP/1.1 200 OK
  < Server: nginx/1.25.4
  < Date: Wed, 03 Jul 2024 00:58:37 GMT
  < Content-Type: text/plain
  < Content-Length: 294
  < Connection: keep-alive
  <
  Headers:
    header 'My-list-of-bands' is 'Led-Zeppelin,Bee-Gees,Simply-Red'
    header 'My-favorite-band-today' is 'Stryper'
    header 'Host' is 'whatigot.example.com'
    header 'X-Forwarded-For' is '10.1.10.11'
    header 'Connection' is 'close'
    header 'Accept' is '*/*'
  * Connection #0 to host whatigot.lab.f5npi.net left intact
  ```

</details>

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments whatigot
kubectl delete service whatigot
kubectl delete gateways gateway
kubectl delete httproutes whatigot
kubectl delete configmap whatigot-config
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [What I Got POD, Service and config map](whatigot.yaml)
- [What I Got Gateway](gateway.yaml)
- [What I Got HTTPRoute](whatigot-httpRoute.yaml)

Previous: [SDE NGINX Gateway Fabric](../README.md)

Next: [Interactive Lab](../lab/README.md)

---

## End of lab
