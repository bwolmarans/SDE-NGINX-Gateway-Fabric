# Fix it lab: Coffee and Tea traffic splitting

## Introduction

In this demo we are going to fix a broken deployment that should be splitting traffic for the [Coffee brands application and services](coffee.yaml) to the default namespace. This Coffee application actually creates three deployments, one called **folgers**, another called **starbucks** and the last one called **peets**. The gateway object and HTTPRoute will be deployed to the default namespace also. 

Copy and paste the following code snippet to deploy the coffee application and service.

```yaml
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: folgers
spec:
  replicas: 1
  selector:
    matchLabels:
      app: folgers
  template:
    metadata:
      labels:
        app: folgers
    spec:
      containers:
      - name: folgers
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: folgers
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: folgers
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: starbucks
spec:
  replicas: 1
  selector:
    matchLabels:
      app: starbucks
  template:
    metadata:
      labels:
        app: starbucks
    spec:
      containers:
      - name: starbucks
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: starbucks
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: starbucks
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: peets
spec:
  replicas: 1
  selector:
    matchLabels:
      app: peets
  template:
    metadata:
      labels:
        app: peets
    spec:
      containers:
      - name: peets
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: peets
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: peets
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
  name: coffee-httproute
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
        value: /coffee
    backendRefs:
    - name: peets
      port: 80
    - name: starbucks
      port: 80
EOF
```

The customers have noticed that the Coffee brands application is not quite working as expected. They would like 80% of the requests to be routed to peets and the other 20% to starbucks. Currently it seems the routing is about 50/50 between peets and starbucks and no one can see the rule in the configuration showing a 50/50 split.

In addition they would like a would any request with the header value secret:coffee to return the folgers application.

The Infrastructure administrators, Cluster administrators and Application developers all claim to have deployed and configured their objects with no errors and cannot explain at this time why things are not working as expected.  They have called you in to help resolve this issue.

You can start with a quick check for the pods and services and then move on from there...

```bash
kubectl get all -owide
```

Next test the basic functionality for /coffee. and passing in a header value.

```bash
curl http://coffee.lab.f5npi.net/coffee
```

The http Route rule using the header value secret:coffee has not been created as is something you will need to create now.  Check the lab samples for clues or review the [gateway api docs](https://gateway-api.sigs.k8s.io/guides/http-routing/)

```bash
curl -H "secret:coffee" http://coffee.lab.f5npi.net/coffee
```

## Clean up

When done with this lab you can clean up the objects by running the following commands.

>**Note**: Replace [TAB][TAB] with actually pressing the tab key to see the object names to remove.

```bash
kubectl delete deployments.apps folgers peets starbucks
kubectl delete service folgers peets starbucks
kubectl delete gateways cafe-gateway
kubectl delete httproutes.gateway coffee-httproute [TAB][TAB]
```

>**Note**: If you would prefer to manually create and deploy the configurations you can find the source YAML files here:

- [Coffee and tea applications and services](coffee-and-tea.yaml)
- [Cafe Gateway](cafe-gateway.yaml)
- [cafe-httpRoute](cafe-httpRoute.yaml)

<details>
  <summary><h3><b>Solution</b><h3></summary>

  [Click here](cafe-httpRoute-solution.yaml) to see a solution for a <b>HTTPRoute weighting</b> configuration.

  [Click here](cafe-httpRoute-secret.yaml) to see a solution for a <b>HTTPRoute headers</b> configuration.
</details>

Return: [SDE NGINX Gateway Fabric](../README.md)

Previous: [Interactive Lab](../lab/README.md)

Return to the: [NGINX Gateway Fabric home page](../../README.md)

---

## End of lab
