# HTTPS Termination Fix It Lab

In this FixIt, try not to look too closely at the sample code and just copy and run it.  This
should result in unexpected behavior.  Your challenge is to find and fix the problem and resolve
the issue.

## Introduction

Salvadore has had some customers comment that they could not find Bag Balm. Salvadore explained that this is just the brand-name equivalent of his Lanolin salve. He asked one of your teammates to make the change during a nighttime maintenance window. Your teammate hired a friend who works in the nighttime cleaning crew who said he used to work with OS2, so he should be pretty good, to make the change. It seemed simple enough, but something went wrong, and the site was not working as expected. Your teammate is moonlighting at Salvadore's archrival, Oly's Ointment Emporium. Oly doesn't allow outside food, drinks, or calls, so you are on your own to figure this out.

What Salvadore wanted was if a user requested <https://salve.lab.f5npi.net/bagbalm> it would send then to the /lanolin URI.

## Fix it lab

## Deploy the backend servers in a UDF to research the fix

Deploy the application first in your UDF lab so you can create the NGF config that is not working and look for the issue.
*This deployment app deployment is not broken*

```bash
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lanolin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lanolin
  template:
    metadata:
      labels:
        app: lanolin
    spec:
      containers:
      - name: lanolin
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: lanolin
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: lanolin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: arnica
spec:
  replicas: 1
  selector:
    matchLabels:
      app: arnica
  template:
    metadata:
      labels:
        app: arnica
    spec:
      containers:
      - name: arnica
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: arnica
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: arnica
EOF
```

### Deploy the the config steps below that your teammate gave the the night crew cleaning person

#### Create the Namespace `certificate` and a Secret with a TLS certificate and key

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: certificate
---
apiVersion: v1
kind: Secret
metadata:
  name: cafe-secret
  namespace: certificate
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUNzakNDQVpvQ0NRQzdCdVdXdWRtRkNEQU5CZ2txaGtpRzl3MEJBUXNGQURBYk1Sa3dGd1lEVlFRRERCQmoKWVdabExtVjRZVzF3YkdVdVkyOXRNQjRYRFRJeU1EY3hOREl4TlRJek9Wb1hEVEl6TURjeE5ESXhOVEl6T1ZvdwpHekVaTUJjR0ExVUVBd3dRWTJGbVpTNWxlR0Z0Y0d4bExtTnZiVENDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFECmdnRVBBRENDQVFvQ2dnRUJBTHFZMnRHNFc5aStFYzJhdnV4Q2prb2tnUUx1ek10U1Rnc1RNaEhuK3ZRUmxIam8KVzFLRnMvQVdlS25UUStyTWVKVWNseis4M3QwRGtyRThwUisxR2NKSE50WlNMb0NEYUlRN0Nhck5nY1daS0o4Qgo1WDNnVS9YeVJHZjI2c1REd2xzU3NkSEQ1U2U3K2Vab3NPcTdHTVF3K25HR2NVZ0VtL1Q1UEMvY05PWE0zZWxGClRPL051MStoMzROVG9BbDNQdTF2QlpMcDNQVERtQ0thaEROV0NWbUJQUWpNNFI4VERsbFhhMHQ5Z1o1MTRSRzUKWHlZWTNtdzZpUzIrR1dYVXllMjFuWVV4UEhZbDV4RHY0c0FXaGRXbElweHlZQlNCRURjczN6QlI2bFF1OWkxZAp0R1k4dGJ3blVmcUVUR3NZdWxzc05qcU95V1VEcFdJelhibHhJZVVDQXdFQUFUQU5CZ2txaGtpRzl3MEJBUXNGCkFBT0NBUUVBcjkrZWJ0U1dzSnhLTGtLZlRkek1ISFhOd2Y5ZXFVbHNtTXZmMGdBdWVKTUpUR215dG1iWjlpbXQKL2RnWlpYVE9hTElHUG9oZ3BpS0l5eVVRZVdGQ2F0NHRxWkNPVWRhbUloOGk0Q1h6QVJYVHNvcUNOenNNLzZMRQphM25XbFZyS2lmZHYrWkxyRi8vblc0VVNvOEoxaCtQeDljY0tpRDZZU0RVUERDRGh1RUtFWXcvbHpoUDJVOXNmCnl6cEJKVGQ4enFyM3paTjNGWWlITmgzYlRhQS82di9jU2lyamNTK1EwQXg4RWpzQzYxRjRVMTc4QzdWNWRCKzQKcmtPTy9QNlA0UFlWNTRZZHMvRjE2WkZJTHFBNENCYnExRExuYWRxamxyN3NPbzl2ZzNnWFNMYXBVVkdtZ2todAp6VlZPWG1mU0Z4OS90MDBHUi95bUdPbERJbWlXMGc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2UUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRQzZtTnJSdUZ2WXZoSE4KbXI3c1FvNUtKSUVDN3N6TFVrNExFeklSNS9yMEVaUjQ2RnRTaGJQd0ZuaXAwMFBxekhpVkhKYy92TjdkQTVLeApQS1VmdFJuQ1J6YldVaTZBZzJpRU93bXF6WUhGbVNpZkFlVjk0RlAxOGtSbjl1ckV3OEpiRXJIUncrVW51L25tCmFMRHF1eGpFTVBweGhuRklCSnYwK1R3djNEVGx6TjNwUlV6dnpidGZvZCtEVTZBSmR6N3Rid1dTNmR6MHc1Z2kKbW9RelZnbFpnVDBJek9FZkV3NVpWMnRMZllHZWRlRVJ1VjhtR041c09va3R2aGxsMU1udHRaMkZNVHgySmVjUQo3K0xBRm9YVnBTS2NjbUFVZ1JBM0xOOHdVZXBVTHZZdFhiUm1QTFc4SjFINmhFeHJHTHBiTERZNmpzbGxBNlZpCk0xMjVjU0hsQWdNQkFBRUNnZ0VBQnpaRE50bmVTdWxGdk9HZlFYaHRFWGFKdWZoSzJBenRVVVpEcUNlRUxvekQKWlV6dHdxbkNRNlJLczUyandWNTN4cU9kUU94bTNMbjNvSHdNa2NZcEliWW82MjJ2dUczYnkwaVEzaFlsVHVMVgpqQmZCcS9UUXFlL2NMdngvSkczQWhFNmJxdFRjZFlXeGFmTmY2eUtpR1dzZk11WVVXTWs4MGVJVUxuRmZaZ1pOCklYNTlSOHlqdE9CVm9Sa3hjYTVoMW1ZTDFsSlJNM3ZqVHNHTHFybmpOTjNBdWZ3ZGRpK1VDbGZVL2l0K1EvZkUKV216aFFoTlRpNVFkRWJLVStOTnYvNnYvb2JvandNb25HVVBCdEFTUE05cmxFemIralQ1WHdWQjgvLzRGY3VoSwoyVzNpcjhtNHVlQ1JHSVlrbGxlLzhuQmZ0eVhiVkNocVRyZFBlaGlPM1FLQmdRRGlrR3JTOTc3cjg3Y1JPOCtQClpoeXltNXo4NVIzTHVVbFNTazJiOTI1QlhvakpZL2RRZDVTdFVsSWE4OUZKZnNWc1JRcEhHaTFCYzBMaTY1YjIKazR0cE5xcVFoUmZ1UVh0UG9GYXRuQzlPRnJVTXJXbDVJN0ZFejZnNkNQMVBXMEg5d2hPemFKZUdpZVpNYjlYTQoybDdSSFZOcC9jTDlYbmhNMnN0Q1lua2Iwd0tCZ1FEUzF4K0crakEyUVNtRVFWNXA1RnRONGcyamsyZEFjMEhNClRIQ2tTazFDRjhkR0Z2UWtsWm5ZbUt0dXFYeXNtekJGcnZKdmt2eUhqbUNYYTducXlpajBEdDZtODViN3BGcVAKQWxtajdtbXI3Z1pUeG1ZMXBhRWFLMXY4SDNINGtRNVl3MWdrTWRybVJHcVAvaTBGaDVpaGtSZS9DOUtGTFVkSQpDcnJjTzhkUVp3S0JnSHA1MzRXVWNCMVZibzFlYStIMUxXWlFRUmxsTWlwRFM2TzBqeWZWSmtFb1BZSEJESnp2ClIrdzZLREJ4eFoyWmJsZ05LblV0YlhHSVFZd3lGelhNcFB5SGxNVHpiZkJhYmJLcDFyR2JVT2RCMXpXM09PRkgKcmppb21TUm1YNmxhaDk0SjRHU0lFZ0drNGw1SHhxZ3JGRDZ2UDd4NGRjUktJWFpLZ0w2dVJSSUpBb0dCQU1CVApaL2p5WStRNTBLdEtEZHUrYU9ORW4zaGxUN3hrNXRKN3NBek5rbWdGMU10RXlQUk9Xd1pQVGFJbWpRbk9qbHdpCldCZ2JGcXg0M2ZlQ1Z4ZXJ6V3ZEM0txaWJVbWpCTkNMTGtYeGh3ZEVteFQwVit2NzZGYzgwaTNNYVdSNnZZR08KditwVVovL0F6UXdJcWZ6dlVmV2ZxdStrMHlhVXhQOGNlcFBIRyt0bEFvR0FmQUtVVWhqeFU0Ym5vVzVwVUhKegpwWWZXZXZ5TW54NWZyT2VsSmRmNzlvNGMvMHhVSjh1eFBFWDFkRmNrZW96dHNpaVFTNkN6MENRY09XVWxtSkRwCnVrdERvVzM3VmNSQU1BVjY3NlgxQVZlM0UwNm5aL2g2Tkd4Z28rT042Q3pwL0lkMkJPUm9IMFAxa2RjY1NLT3kKMUtFZlNnb1B0c1N1eEpBZXdUZmxDMXc9Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K
EOF
```

#### Create the ReferenceGrant to allow the Gateway resource in the default Namespace to access the certificate Secret in the Namespace named **certificate**

```shell
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: access-cert-secret
  namespace: certificate
spec:
  to:
  - group: ""
    kind: Secret
    name: cafe-secret
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: default
EOF
```

#### Create the Gateway resource

```shell
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: salve-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: cafe-secret
        namespace: certificate
EOF
```

#### Create the HTTPRoute resources

```shell
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: salvadores-tls-redirect
spec:
  parentRefs:
  - name: salve-gateway
    sectionName: http
  hostnames:
  - "salve.lab.f5npi.net"
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        port: 443
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: lanolin
spec:
  parentRefs:
  - name: salve-gateway
    sectionName: https
  hostnames:
  - "salve.lab.f5npi.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /lanolin
    backendRefs:
    - name: lanolin
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: arnica
spec:
  parentRefs:
  - name: salve-gateway
    sectionName: https
  hostnames:
  - "salve.lab.f5npi.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /arnica
    backendRefs:
    - name: arnica
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bagbalm
spec:
  parentRefs:
  - name: bagbalm-gateway
    sectionName: https
  hostnames:
  - "salve.lab.f5npi.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /bagbalm
    backendRefs:
    - name: lanolin
      port: 80
---
EOF
```

## Problem description

You get to work and do not want to expose your teammate for double dipping because they owe you money. You decide to see if the /lanolin and /arnica sites are working and then check /bagbalm.

First you check the TLS redirect to make sure it is working by making an HTTP request to the lanolin site with -I to see the headers.

#### Check redirect

```shell
curl -I  http://salve.lab.f5npi.net/lanolin

```

<details>
  <summary><h3>Redirect test response working correctly</h3></summary>

  ```bash
  f5admin@bastion:~$ curl -I  http://salve.lab.f5npi.net/lanolin
  HTTP/1.1 302 Moved Temporarily
  Server: nginx/1.27.0
  Date: Fri, 12 Jul 2024 20:49:56 GMT
  Content-Type: text/html
  Content-Length: 145
  Connection: keep-alive
  Location: https://salve.lab.f5npi.net/lanolin
  ```

</details>

#### Check /lanolin

You check HTTPS to the /lanolin path:

```shell
curl -k  https://salve.lab.f5npi.net/lanolin
```

<details>
  <summary><h3>Successful request to https://salve.lab.f5npi.net/lanolin</h3></summary>

  ```shell
  f5admin@bastion:~$ curl -k  https://salve.lab.f5npi.net/lanolin
  Server address: 10.244.67.147:8080
  Server name: lanolin-545d67bfb-wqcg9
  Date: 12/Jul/2024:20:55:46 +0000
  URI: /lanolin
  Request ID: 65895c0fcaec730e3506ba52237def3e
  ```

</details>

#### Check /arnica

You check HTTPS to the /arnica path:

```shell
curl -k  https://salve.lab.f5npi.net/arnica
```

<details>
  <summary><h3>Successful request to https://salve.lab.f5npi.net/arnica</h3></summary>

  ```shell
  f5admin@bastion:~$ curl -k  https://salve.lab.f5npi.net/arnica
  Server address: 10.244.191.143:8080
  Server name: arnica-6c54c79b6b-cccx6
  Date: 12/Jul/2024:21:17:56 +0000
  URI: /arnica
  Request ID: 92b8adc5378c01048ea1cdb55c218f02
  ```

</details>

#### Check /bagbalm

```shell
curl -k  https://salve.lab.f5npi.net/bagbalm
```

<details>
  <summary><h3>Failing request to https://salve.lab.f5npi.net/bagbalm</h3></summary>

  ```shell
  f5admin@bastion:~$ curl -k  https://salve.lab.f5npi.net/bagbalm
  <html>
  <head><title>404 Not Found</title></head>
  <body>
  <center><h1>404 Not Found</h1></center>
  <hr><center>nginx/1.27.0</center>
  </body>
  </html>
  ```

</details>

#### What working looks like

What you should see when going to /bagbalm is the exact same response you would expect from /lanolin since they are routed to the same app.

<details>
  <summary><h3>HINT</h3></summary>

  What is the name of the gateway used in the new rule.

</details>

Once you have fixed this you better clean up and go home because you got a message on Teams from another teammate complaining about the sandwich vending machine guy screwed of the config on the firewall. You better get out of there before they show up at your cubicle.

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments arnica lanolin
kubectl delete services arnica lanolin
kubectl delete namespaces certificate
kubectl delete gateways salve-gateway
kubectl delete httproutes salvadores-tls-redirect lanolin arnica bagbalm
```

Return: [SDE NGINX Gateway Fabric](../README.md)

Previous: [Interactive Lab](../lab/README.md)

Next: [Advanced Routing Use Case](../../use-case5-and-6-advanced-routing-traffic-splitting/README.md)

---

## End of lab
