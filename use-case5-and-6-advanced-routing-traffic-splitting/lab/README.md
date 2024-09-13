# Advanced Routing and Traffic Splitting Lab

This lab is intended to be completed by individuals or small groups during the Up skilling sessions
but it maybe completed at anytime.

## Introduction

This lab was created based on the following *How-To* guide below.
<https://docs.nginx.com/nginx-gateway-fabric/how-to/traffic-management/advanced-routing/>

This lab covers two use cases:

1. Advanced Routing
2. Traffic Splitting

For the first part, the Coffee Shop needs traffic routed depending on the HTTP method (**GET** or **POST**) that is
used. A **GET** request is routed to a special **get** server and the **POST** is routed to a special
**post** server. You will configure an **HTTPRoute** for this scenario.

For the second part, the same Coffee Shop needs to handle backend server upgrades. Both servers
have a version update but the owner is not ready to fully release it to production fearing
downtime. This person wants to be confident the new **v2** version will not cause issues so the
person wants to only use the new backend application version based on the headers and query
parameters for testing. Existing traffic should still be going to the current **v1** version. You
will update the existing configuration to allow this scenario to work.

For the third part, the owner feels good to proceed to the next step after testing from part 2.
The person wants to take one more step of being diligent and split traffic between the existing
**v1** application and the and new **v2** application. You will update the configuration to enable
traffic splitting.

Below is a summary of the intent of this lab.

1. Configure a route such that a **GET** will be routed to a **get** server and a **POST** will be
   routed to a **post** server.
2. Update the route such that traffic goes to this newly released version, **v2**, based on either
   a header value or a query parameter.
3. Finally, update the route such that traffic get split between these two versions where 50% goes
   to the old **v1** version and 50% goes to the new **v2** version.

## Part 1: advanced routing

1. In the first part, we will deploy all backend applications. This part of the lab only uses the
   **v1** backend applications.

    Copy and paste the following code snippet to deploy the app.

    > If you prefer to manually create this, click [here](coffee.yaml) for the YAML file.

    ```bash
    kubectl create -f - <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: coffee-v1-get
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: coffee-v1-get
      template:
        metadata:
          labels:
            app: coffee-v1-get
        spec:
          containers:
          - name: coffee-v1-get
            image: nginxdemos/nginx-hello:plain-text
            ports:
            - containerPort: 8080
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: coffee-v1-get-svc
    spec:
      ports:
      - port: 80
        targetPort: 8080
        protocol: TCP
        name: http
      selector:
        app: coffee-v1-get
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: coffee-v1-post
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: coffee-v1-post
      template:
        metadata:
          labels:
            app: coffee-v1-post
        spec:
          containers:
          - name: coffee-v1-post
            image: nginxdemos/nginx-hello:plain-text
            ports:
            - containerPort: 8080
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: coffee-v1-post-svc
    spec:
      ports:
      - port: 80
        targetPort: 8080
        protocol: TCP
        name: http
      selector:
        app: coffee-v1-post
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: coffee-v2-get
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: coffee-v2-get
      template:
        metadata:
          labels:
            app: coffee-v2-get
        spec:
          containers:
          - name: coffee-v2-get
            image: nginxdemos/nginx-hello:plain-text
            ports:
            - containerPort: 8080
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: coffee-v2-get-svc
    spec:
      ports:
      - port: 80
        targetPort: 8080
        protocol: TCP
        name: http
      selector:
        app: coffee-v2-get
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: coffee-v2-post
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: coffee-v2-post
      template:
        metadata:
          labels:
            app: coffee-v2-post
        spec:
          containers:
          - name: coffee-v2-post
            image: nginxdemos/nginx-hello:plain-text
            ports:
            - containerPort: 8080
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: coffee-v2-post-svc
    spec:
      ports:
      - port: 80
        targetPort: 8080
        protocol: TCP
        name: http
      selector:
        app: coffee-v2-post
    EOF
    ```

    Now check your new pods and services.

    ```bash
    kubectl get pods,services -owide
    kubectl get deployments
    ```

    <details>
      <summary><b>Example output</b></summary>

      ```bash
      f5admin@bastion:~$ kubectl get pods,services -owide
      NAME                                  READY   STATUS    RESTARTS   AGE     IP               NODE                    NOMINATED NODE   READINESS GATES
      pod/coffee-v1-get-54c5ccd8f6-xdm22    1/1     Running   0          2m31s   10.244.191.150   w3-mgmt.lab.f5npi.net   <none>           <none>
      pod/coffee-v1-post-6449994dbd-z5s25   1/1     Running   0          2m31s   10.244.82.166    w2-mgmt.lab.f5npi.net   <none>           <none>
      pod/coffee-v2-get-574bd867b8-dvm59    1/1     Running   0          2m31s   10.244.191.151   w3-mgmt.lab.f5npi.net   <none>           <none>
      pod/coffee-v2-post-8f5974798-ls4wn    1/1     Running   0          2m31s   10.244.67.161    w1-mgmt.lab.f5npi.net   <none>           <none>

      NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
      service/coffee-v1-get-svc    ClusterIP   10.97.159.168    <none>        80/TCP    2m31s   app=coffee-v1-get
      service/coffee-v1-post-svc   ClusterIP   10.96.167.157    <none>        80/TCP    2m31s   app=coffee-v1-post
      service/coffee-v2-get-svc    ClusterIP   10.106.48.59     <none>        80/TCP    2m31s   app=coffee-v2-post
      service/coffee-v2-post-svc   ClusterIP   10.105.168.252   <none>        80/TCP    2m30s   app=coffee-v2-post
      service/kubernetes           ClusterIP   10.96.0.1        <none>        443/TCP   16d     <none>
      f5admin@bastion:~$ kubectl get deployments
      NAME             READY   UP-TO-DATE   AVAILABLE   AGE
      coffee-v1-get    1/1     1            1           2m36s
      coffee-v1-post   1/1     1            1           2m36s
      coffee-v2-get    1/1     1            1           2m36s
      coffee-v2-post   1/1     1            1           2m36s
      ```

    </details>

2. Next, create an NGINX Gateway Fabric resource that will enable traffic into NGF based
   on the following criteria.

    | Listener Name | Protocol       | Port      | hostname              |
    | --------------| -------------- | --------- | --------------------- |
    | `http`        |`HTTP`          | `80`      |  `*.lab.f5npi.net`    |

    This could be the responsibility of the **Cluster Administrator**.

    Copy and paste the following code snippet to deploy the gateway.

    > If you prefer to manually create this, click [here](coffee-gateway.yaml) for the YAML file.

    ```bash
    kubectl create -f - <<EOF
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: cafe
    spec:
      gatewayClassName: nginx
      listeners:
      - name: http
        port: 80
        protocol: HTTP
        hostname: "*.lab.f5npi.net"
    EOF
    ```

    Check the new **cafe** Gateway health.

    ```bash
    kubectl get gateways cafe
    kubectl describe gateways cafe
    ```

    <details>
      <summary><b>Example output</b></summary>

      ```bash
      f5admin@bastion:~$ kubectl get gateways cafe
    NAME   CLASS   ADDRESS       PROGRAMMED   AGE
    cafe   nginx   10.1.10.100   True         2m23s
    f5admin@bastion:~$ kubectl describe gateways cafe
    Name:         cafe
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    API Version:  gateway.networking.k8s.io/v1
    Kind:         Gateway
    Metadata:
      Creation Timestamp:  2024-07-22T06:10:41Z
      Generation:          1
      Resource Version:    260821
      UID:                 05dcdf02-4180-4b4f-a7d6-8a815578e127
    Spec:
      Gateway Class Name:  nginx
      Listeners:
        Allowed Routes:
          Namespaces:
            From:  Same
        Hostname:  *.lab.f5npi.net
        Name:      http
        Port:      80
        Protocol:  HTTP
    Status:
      Addresses:
        Type:   IPAddress
        Value:  10.1.10.100
      Conditions:
        Last Transition Time:  2024-07-22T06:10:41Z
        Message:               Gateway is accepted
        Observed Generation:   1
        Reason:                Accepted
        Status:                True
        Type:                  Accepted
        Last Transition Time:  2024-07-22T06:10:41Z
        Message:               Gateway is programmed
        Observed Generation:   1
        Reason:                Programmed
        Status:                True
        Type:                  Programmed
      Listeners:
        Attached Routes:  0
        Conditions:
          Last Transition Time:  2024-07-22T06:10:41Z
          Message:               Listener is accepted
          Observed Generation:   1
          Reason:                Accepted
          Status:                True
          Type:                  Accepted
          Last Transition Time:  2024-07-22T06:10:41Z
          Message:               Listener is programmed
          Observed Generation:   1
          Reason:                Programmed
          Status:                True
          Type:                  Programmed
          Last Transition Time:  2024-07-22T06:10:41Z
          Message:               All references are resolved
          Observed Generation:   1
          Reason:                ResolvedRefs
          Status:                True
          Type:                  ResolvedRefs
          Last Transition Time:  2024-07-22T06:10:41Z
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

3. Create an HTTPRoute resource for the coffee application. The expected behavior is summarized in
   the table below.

    | Method       | hostname             | Path         | Backend Service      |
    | ------------ | -------------------- | ------------ | -------------------- |
    | `GET`        | `cafe.lab.f5npi.net` | `/coffee`    | `coffee-v1-get-svc`  |
    | `POST`       | `cafe.lab.f5npi.net` | `/coffee`    | `coffee-v1-post-svc` |

    However, to make this lab just slightly challenging the following code snippet will create a file
    called **coffee-httpRoute-part1.yaml** but it will not work until you update the values surrounded in
    **##** symbols.

    ```bash
    echo "apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: coffee-part1
    spec:
      parentRefs:
      - name: cafe
      hostnames:
      - cafe.lab.f5npi.net
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: ##Path##
          method: ##First Method##
        backendRefs:
        - name: ##Fist Backend Application Service##
          port: 80
      - matches:
        - path:
            type: PathPrefix
            value: ##Path##
          method: ##Second Method##
        backendRefs:
        - name: ##Second Backend Application Service##
          port: 80" > coffee-httpRoute-part1.yaml
    ```

    Once you have fixed the YAML configuration file, create the object using the command below.

    ```bash
    kubectl create -f coffee-httpRoute-part1.yaml
    ```

## Test the application part 1

Run the following commands to confirm you are able to successfully pass traffic to each
backend application.

```bash
curl cafe.lab.f5npi.net/coffee
curl cafe.lab.f5npi.net/coffee -X POST
curl cafe.lab.f5npi.net/coffee -X OPTIONS
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee
  Server address: 10.244.191.150:8080
  Server name: coffee-v1-get-54c5ccd8f6-xdm22
  Date: 22/Jul/2024:06:28:07 +0000
  URI: /coffee
  Request ID: c8d00dc25d8fcdfd1382de90fc3491c3

  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X POST
  Server address: 10.244.82.166:8080
  Server name: coffee-v1-post-6449994dbd-z5s25
  Date: 22/Jul/2024:06:28:52 +0000
  URI: /coffee
  Request ID: c68923a58e34f5f5d3bf2b6c1443b861

  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X OPTIONS
  <html>
  <head><title>404 Not Found</title></head>
  <body>
  <center><h1>404 Not Found</h1></center>
  <hr><center>nginx/1.27.0</center>
  </body>
  </html>
  ```

</details>

**Question**: For these 3 requests, which **HTTPRoute** rules match which request? Why?

<details>
  <summary><b>Answer</b></summary>

```bash
curl cafe.lab.f5npi.net/coffee
```

This by default `curl` uses the **GET** method. And this request matches the rule section that
corresponds to the rule that defines the **GET** method.

```bash
curl cafe.lab.f5npi.net/coffee -X POST
```

This `curl` request sends a **POST**. And matches the rule section that corresponds to the section
that defines the **POST** rule.

```bash
curl cafe.lab.f5npi.net/coffee -X OPTIONS
```

This returns a 404 because it matches nothing. There is no rule looking for the method **OPTIONS**.
NGF does not find any resource to handle the request and returns a **404 Not Found**.

</details>

<details>
  <summary><h3>Solution</h3></summary>

  Click [here](coffee-httpRoute-part1.yaml) to see the HTTPRoute solution for part 1.
</details>

## Part 2: advanced routing

Some time has gone by and there is a major release to the backend servers. The owner of Coffee
Shop is not fully ready to switch over to the new version quite yet. The person wants to spend time
testing the new version by only allowing traffic the new applications version shown in the table
below.

| Field         | Name       | Value     |
| ------------- | ---------- | --------- |
| header        | `version`  | `v2`      |
| queryParams   | `TEST`     | `v2`      |

Update the existing **HTTPRoute** resource to account for the behavior above..

Use the [Kubernetes HTTPRouteMatch Reference](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.HTTPRouteMatch)
to see how to format the YAML file.

In the case here, you will want to also look at the following references.

1. [Kubernetes HTTPHeaderMatch Reference](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.HTTPHeaderMatch)
2. [Kubernetes HTTPQueryParamMatch Reference](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPQueryParamMatch)

## Part 2: Test the application

Now test this new change by running the commands below.

The below commands should route traffic to the **v1** servers.

```bash
curl cafe.lab.f5npi.net/coffee
curl cafe.lab.f5npi.net/coffee -X POST
```

And the below commands should route traffic to the **v2** servers.

```bash
curl cafe.lab.f5npi.net/coffee -H "version:v2"
curl cafe.lab.f5npi.net/coffee -X POST -H "version:v2"
curl cafe.lab.f5npi.net/coffee?TEST=v2
curl cafe.lab.f5npi.net/coffee?TEST=v2 -X POST
```

<details>
  <summary><b>Example output</b></summary>

  Below is the example output for each command being tested. Notice the **Server name** to
  differentiate between the **get** or **post** servers. And **v1** and **v2** to differentiate
  the different versions.

  The output below show traffic going to the **v1** servers.

  ```bash
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee
  Server address: 10.244.191.150:8080
  Server name: coffee-v1-get-54c5ccd8f6-xdm22
  Date: 22/Jul/2024:07:31:14 +0000
  URI: /coffee
  Request ID: 8782c6bc65b7b3f73384d08e66fc0b57

  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X POST
  Server address: 10.244.82.166:8080
  Server name: coffee-v1-post-6449994dbd-z5s25
  Date: 22/Jul/2024:07:31:19 +0000
  URI: /coffee
  Request ID: e1ccc4661d968b1bb5dbbef4974ee079
  ```

  Any the commands below show traffic going to the **v2** servers.

  ```bash
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -H "version:v2"
  Server address: 10.244.191.151:8080
  Server name: coffee-v2-get-574bd867b8-dvm59
  Date: 22/Jul/2024:07:31:23 +0000
  URI: /coffee
  Request ID: 8435ab2b33137fadd14e24dac8535f5a

  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X POST -H "version:v2"
  Server address: 10.244.67.161:8080
  Server name: coffee-v2-post-8f5974798-ls4wn
  Date: 22/Jul/2024:07:31:30 +0000
  URI: /coffee
  Request ID: cd373065f94bc55ef73a9f6af785e6c6

  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee?TEST=v2
  Server address: 10.244.191.151:8080
  Server name: coffee-v2-get-574bd867b8-dvm59
  Date: 22/Jul/2024:07:31:35 +0000
  URI: /coffee?TEST=v2
  Request ID: ed05d215c5b54b301ab8c8c86392bf21

  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee?TEST=v2 -X POST
  Server address: 10.244.67.161:8080
  Server name: coffee-v2-post-8f5974798-ls4wn
  Date: 22/Jul/2024:07:31:40 +0000
  URI: /coffee?TEST=v2
  Request ID: b00c4c07998bed54d3e0f0129d7a816a
  ```

</details>

<details>
  <summary><h3>Solution</h3></summary>

  Click [here](coffee-httpRoute-part2.yaml) to see the HTTPRoute solution for part 2.
</details>

## Part 3: Traffic splitting

Now we are on to the final part of this lab. We want to update the existing **HTTPRoute** such that
traffic is split 50/50 between versions **v1** and **v2**.

We also want to keep the existing behavior from part 2 where we can target the new **v2** servers
and also include splitting traffic.

You may want to reference below to find the field needed to split traffic between two backend
applications.

1. [Kubernetes BackendRef Reference](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.BackendRef)

## Part 3: Test the application

When you run the following commands multiple times, we expect to see traffic going to both **v1**
and **v2** about half the time.

```bash
curl cafe.lab.f5npi.net/coffee
```

The commands below should continue to the **v2** servers 100% of the time.

```bash
curl cafe.lab.f5npi.net/coffee -H "version:v2"
curl cafe.lab.f5npi.net/coffee -X POST -H "version:v2"
curl cafe.lab.f5npi.net/coffee?TEST=v2
curl cafe.lab.f5npi.net/coffee?TEST=v2 -X POST
```

<details>
  <summary><b>Example output</b></summary>

  Below is the example output for each command being testing. Notice the **Server name** to
  differentiate between the **get** or **post** servers. And **v1** and **v2** to differentiate
  the different versions. Also notice the **Server name** show response from **v1** and **v2**.

  The output from the **get** server is shown below.

  ```bash
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee
  Server address: 10.244.191.150:8080
  Server name: coffee-v1-get-54c5ccd8f6-xdm22
  Date: 22/Jul/2024:07:32:39 +0000
  URI: /coffee
  Request ID: abd40efa16f5ad3cab918701fafc1d93
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee
  Server address: 10.244.191.150:8080
  Server name: coffee-v1-get-54c5ccd8f6-xdm22
  Date: 22/Jul/2024:07:32:40 +0000
  URI: /coffee
  Request ID: 9b5ac63196d9d58ae505b5521afb26d4
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee
  Server address: 10.244.191.151:8080
  Server name: coffee-v2-get-574bd867b8-dvm59
  Date: 22/Jul/2024:07:32:40 +0000
  URI: /coffee
  Request ID: 7532d13a9bf1c3749eb4b34bd92ea728
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee
  Server address: 10.244.191.150:8080
  Server name: coffee-v1-get-54c5ccd8f6-xdm22
  Date: 22/Jul/2024:07:32:41 +0000
  URI: /coffee
  Request ID: 580faf8dfaf65258605ad293be9e35b6
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee
  Server address: 10.244.191.151:8080
  Server name: coffee-v2-get-574bd867b8-dvm59
  Date: 22/Jul/2024:07:32:42 +0000
  URI: /coffee
  Request ID: fb4f04e7572828f883ee72249ead656e
  ```

  The output from the **post** server is shown below.

  ```bash
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X POST
  Server address: 10.244.67.161:8080
  Server name: coffee-v2-post-8f5974798-ls4wn
  Date: 22/Jul/2024:07:34:05 +0000
  URI: /coffee
  Request ID: 17ae55073f7e863bd6c8348ee62d07fa
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X POST
  Server address: 10.244.67.161:8080
  Server name: coffee-v2-post-8f5974798-ls4wn
  Date: 22/Jul/2024:07:34:06 +0000
  URI: /coffee
  Request ID: 1796c59887ac58b59aac2a4eadec9e05
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X POST
  Server address: 10.244.82.166:8080
  Server name: coffee-v1-post-6449994dbd-z5s25
  Date: 22/Jul/2024:07:34:07 +0000
  URI: /coffee
  Request ID: fdd30a1da963558a406a960879827cb1
  f5admin@bastion:~$ curl cafe.lab.f5npi.net/coffee -X POST
  Server address: 10.244.82.166:8080
  Server name: coffee-v1-post-6449994dbd-z5s25
  Date: 22/Jul/2024:07:34:08 +0000
  URI: /coffee
  Request ID: 907dcee4356ac06b2288ab3626e0a22e
  ```

</details>

<details>
  <summary><h3>Solution</h3></summary>

  Click [here](coffee-httpRoute-part3.yaml) to see the HTTPRoute solution for part 3.
</details>

## Clean up

When you are done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments coffee-v1-get coffee-v1-post coffee-v2-get coffee-v2-post
kubectl delete svc coffee-v1-get-svc coffee-v1-post-svc coffee-v2-get-svc coffee-v2-post-svc
kubectl delete gateways coffee
kubectl delete httproutes coffee
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

* [Coffee applications and services](coffee.yaml)
* [Coffee Gateway](coffee-gateway.yaml)
* [HTTPRoute Part 1](coffee-httpRoute-part1.yaml)
* [HTTPRoute Part 2](coffee-httpRoute-part2.yaml)
* [HTTPRoute Part 3](coffee-httpRoute-part3.yaml)

Previous: [Demo](../demo/README.md)

>**Note**: The fixit lab is based on the traffic splitting use case only.

Next: [Fix It Lab](../../bonus-labs/case6-split-traffic-connections/fixit/README.md)

---

## End of lab
