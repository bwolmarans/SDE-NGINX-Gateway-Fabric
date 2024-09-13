# Modify Request Headers Lab

## Introduction

The coffee shop is looking to test out their new mobile app but need to modify the request headers for validation purposes.  In this example we are going to deploy a coffee application and service along with a config map to help with this testing.

In this example we will configure traffic routing for the coffee-mobile container in our coffee POD. We will use HTTPRoute resources to manage the request headers for the clients of the coffee application using the `RequestHeaderModifier` filter to modify headers to the request.

## Interactive Lab

Copy and paste the following code snippet to deploy the **coffee** application,service and config map.  The **coffee** application is providing a web server with an echo function that will respond with the values inserted by the NGINX Gateway Fabric **HTTPRoute** `RequestHeaderModifier` filter.

> If you prefer to manually create and deploy this application, click [here](coffee-mobile.yaml) for the YAML file.

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
      - name: coffee-mobile
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
          name: coffee-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coffee-config
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

Now check the health of your backend application.

```bash
kubectl get pods,services,configmaps -owide
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~/uc3$ kubectl get pods,services,configmaps -owide
  NAME                          READY   STATUS    RESTARTS   AGE    IP               NODE                    NOMINATED NODE   READINESS GATES
  pod/headers-6f854c478-8bpkj   1/1     Running   0          102s   10.244.191.151   w3-mgmt.lab.f5npi.net   <none>           <none>

  NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE     SELECTOR
  service/headers      ClusterIP   10.100.71.7   <none>        80/TCP    102s    app=headers
  service/kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP   5d20h   <none>

  NAME                         DATA   AGE
  configmap/coffee-config     2      102s
  configmap/kube-root-ca.crt   1      5d20h
  ```

</details>

Next we will create the Gateway resource with the following details.

| Name                   | Value     |
| ---------------------- | -------   |
| **gateway name**       | `cafe-gateway` |
| **gatewayClassName**   | `nginx`   |
| Listener **name**      | `http`    |
| Listener **port**      | `80`      |
| Listener **protocol**  | `HTTP`    |

Copy and paste the following code snippet to create the **Gateway**.

> If you prefer to manually create this, click [here](../demo/gateway.yaml) for the YAML file.

```bash
kubectl create -f -<< EOF
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
EOF
```

Soon we will create the rules and attach then to the gateway listener named **http** that we just created.

Finally, create the HTTPRoute resource. We want the rules be evaluated for traffic from the
listeners defined in the gateway above. Below is a table showing the modifications we want to
change on the request header.

| Type    | Header Name             | Value                       |
| ------- | ----------------------- | --------------------------- |
| set     | **User-Agent** | `Mozilla/5.0 (iPhone; CPU iPhone OS 17_5_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1`    |
| add     | **Accept-Encoding**     | `compress`                  |
| add     | **Mobile-coffee-test**      | `qa-udf-lab` |
| remove  | **Accept-Language**      |                 |

The challenge in this lab is to correct the HTTPRoute file that will be created in the next step. The file will be called **coffee-test-httpRoute.yaml** your challenge is to replace the add, set and remove blocks.  The add section includes the name and value keys but the set and remove do not, you will need to add those details.

>**Note**: You might refer to [Kubernetes RequestHeaderModifier](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.HTTPHeaderFilter) for additional tips on how to format the HTTP Header Filters.

```bash
echo "apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: coffee-mobile-httproute
spec:
  parentRefs:
  - name: cafe-gateway
    sectionName: http
  hostnames:
  - "coffee.lab.f5npi.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /menu
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
          - name: ##Header Name##
            value: ##Header Value##
          - name: ##Header Name##
            value: ##Header Value##
        set:
          - name: ##Header Name##
            value: ##Header Value##
        remove:
          - ##Headers to remove##
    backendRefs:
    - name: coffee
      port: 80"> coffee-mobile-httpRoute.yaml
```

Once you have fixed the YAML configuration file, create the object using the command below.

```bash
kubectl create -f coffee-mobile-httpRoute.yaml
```

<details>
  <summary><h3>solution for the coffee-test-httpRoute.yaml </h3></summary>

First RequestHeaderModifier section uses the *set* command

- The `set` command will add or overwrite a header value
  - if the header exists overwrite the value
  - if header does not exists set with value

```yaml
...
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        set:
        - name: User-Agent
          value: Mozilla/5.0 (iPhone; CPU iPhone OS 17_5_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1
...
```

The next command in the RequestHeaderModifier section uses the *add* command

- The *add* will append the value in an existing header
- The *add* will set the header with the value if it does not exists

```
...
        add:
        - name: Accept-Encoding
          value: compress
        - name: Mobile-coffee-test
          value: qa-udf-lab
...
```

The next section of the RequestHeaderModifier filters uses the *remove* command

- If the header is in the request it will be removed.

```
...
        remove:
        - Accept-Language
...
```

</details>

### Details about the HTTPRoute configuration YAML

The HTTPRoute defines the following.

The HTTPRoute uses the parentRefs to attach to the gateway named **cafe-gateway** and the listener named **http** in that gateway.

```yaml
...
spec:
  parentRefs:
  - name: cafe-gateway
    sectionName: http
...
```

We have also defined the hostname as `coffee.lab.f5npi.net` in the **HTTPRoute** file.

```yaml
...
  hostnames:
  - "coffee.lab.f5npi.net"
...
```

In this example we have the PathPrefix set to `/menu`.

```yaml
...
 - matches:
    - path:
        type: PathPrefix
        value: /menu
...
```

The `backendRefs` section will send any traffic handled by the rule to the service named `coffee`
after it has been processed by the HTTPRoute filter.

>**Question**: What is the container name in the coffee POD?

```yaml
...
   backendRefs:
    - name: coffee
      port: 80
...
```

## Test the Application

To access the application, we will use `curl` to send requests to the `headers` Service, including
sending headers with our request.
Notice our configured header values can be seen in the `requestHeaders` section below, and that
the `Accept-Language` header is absent.

```bash
curl -v http://coffee.lab.f5npi.net/menu -H "Accept-Language:en-US, en-CA"
```

<details>
  <summary><h3>Example output</h3></summary>

  ```text
  Headers:
  header 'Accept-Encoding' is 'compress'
  header 'Mobile-coffee-test' is 'qa-udf-lab'
  header 'User-Agent' is 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_5_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1'
  header 'Host' is 'coffee.lab.f5npi.net'
  header 'X-Forwarded-For' is '10.1.10.11'
  header 'Connection' is 'close'
  header 'Accept' is '*/*'
  ```

  In the above output:
<!-- markdownlint-disable MD007 -->
  - 'Accept-Encoding' is `compress`
    - This was added/set by a rule
  - 'User-Agent' is 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_5_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1'
    - This was set by a rule, the original value was replaced.
  - header 'Mobile-coffee-test' is 'qa-udf-lab'
    - This was added by a rule, Mobile-coffee-test is a custom header
  - Accept-Language
    - This standard header was removed
<!-- markdownlint-enable MD007 -->

</details>

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployment coffee
kubectl delete services coffee
kubectl delete configmap coffee-config
kubectl delete gateways cafe-gateway
kubectl delete httproutes  coffee-mobile-httproute
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [What I Got POD and Service](coffee-mobile.yaml)
- [What I Got Config Map](coffee-configmap.yaml)
- [What I Got Gateway](gateway.yaml)

## Solution

- [What I Got HTTPRoute](coffee-mobile-httpRoute.yaml)

Previous: [Demo](../demo/README.md)

Next: [Fix It Lab](../fixit/README.md)

---

## End of lab
