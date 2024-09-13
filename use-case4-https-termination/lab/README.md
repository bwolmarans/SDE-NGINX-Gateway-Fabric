# Salvador's Salve Shop Lab

This lab is intended to be completed by individuals or small groups during the up-skilling sessions; however, it may be completed any time.

## Introduction

We need to secure the client side of Salvador's customer's transaction by adding HTTPS termination to this site.

We will be doing the following in this lab.

- Create a Namespace containing a Secret object for the TLS certificate
- Create a ReferenceGrant to allow the Gateway object to access the certificate from a different Namespace
- Create a Gateway object with listeners for HTTP and HTTPS requests
- Create HTTPRoute rules that reference section of the Gateway object
- Create three HTTPRoute rules
  - Route traffic with hostname **salve.lab.f5npi.net** and path **/lanolin** to the **Lanlin** app service
  - Route traffic with hostname **salve.lab.f5npi.net** and path **/arnica** to the **Arnica** app service
  - Redirect HTTP to HTTPS

## Interactive Lab

1. Prepare the lab environment applications that will provide the backend application servers.

    Copy and paste the following code snippet to deploy the Lanolin and Arnica application and service.

    This might be completed by the **Applications Developer**.

    > If you prefer to manually create and deploy this application, click [here](salve.yaml) for the YAML file.

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

    Check that the Pods are running in the `default` namespace:

    ```bash
    kubectl -n default get pods,services
    ```

    <details>
      <summary><b>Example output</b></summary>

      ```bash
      f5admin@bastion:~$ kubectl -n default get pods,services
      NAME                          READY   STATUS    RESTARTS   AGE
      pod/arnica-6c54c79b6b-4sjwh   1/1     Running   0          3m59s
      pod/lanolin-545d67bfb-tz22v   1/1     Running   0          4m

      NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
      service/arnica       ClusterIP   10.104.235.114   <none>        80/TCP    3m59s
      service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   10d
      service/lanolin      ClusterIP   10.106.197.65    <none>        80/TCP    4m
      ```

    </details>

2. Create a **Secret** that contains TLS certificate and key in it's own namespace. The TLS
   certificate and key in this Secret are used to terminate the TLS connections for the **Salve**
   applications.

    Copy and paste the following snippet to create a secret in a new namespace that contains a TLS certificate and key.

    > If you prefer to manually create and deploy this, click [here](certificate-ns-and-cafe-secret.yaml) for the YAML file.

    ```bash
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

    > **Important**: This certificate and key are for demo purposes only.

    A **Secret** is created with a certificate installed named **cafe-secret** in namespace **certificate**. *This could be the responsibility of the **cluster administrator**. Now verify the new secret in a new namespace.

    ```bash
    kubectl -n certificate get secrets
    kubectl -n certificate describe secrets cafe-secret
    ```

    <details>
      <summary><b>Example output</b></summary>

      ```bash
      f5admin@bastion:~$ kubectl -n certificate get secrets
      NAME          TYPE                DATA   AGE
      cafe-secret   kubernetes.io/tls   2      11m

      f5admin@bastion:~$ kubectl -n certificate describe secrets cafe-secret
      Name:         cafe-secret
      Namespace:    certificate
      Labels:       <none>
      Annotations:  <none>

      Type:  kubernetes.io/tls

      Data
      ====
      tls.crt:  997 bytes
      tls.key:  1704 bytes
      ```

    </details>

3. Create a Gateway resource to enable incoming traffic into NGF based on the following criteria.

    | Listener Name | Protocol       | Port      | TLS                   |
    | --------------| -------------- | --------- | --------------------- |
    | `http`        |`HTTP`          | `80`      |                       |
    | `https`       |`HTTPS`         | `443`     | name: `cafe-secret` namespace: `certificate` |

    Copy and paste the following code snippet to deploy the **salve-gateway** Gateway.

    > If you prefer to manually create this, click [here](salve-gateway.yaml) for the YAML file.

    ```bash
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

    Check the new **salve-gateway** health.

    ```bash
    kubectl get gateways salve-gateway
    kubectl describe gateways salve-gateway
    ```

    <details>
      <summary><b>Example output</b></summary>

      ```bash
      f5admin@bastion:~$ kubectl get gateways salve-gateway
      NAME            CLASS   ADDRESS       PROGRAMMED   AGE
      salve-gateway   nginx   10.1.10.100   True         12m
      f5admin@bastion:~$ kubectl describe gateways salve-gateway
      Name:         salve-gateway
      Namespace:    default
      Labels:       <none>
      Annotations:  <none>
      API Version:  gateway.networking.k8s.io/v1
      Kind:         Gateway
      Metadata:
        Creation Timestamp:  2024-07-16T18:18:41Z
        Generation:          1
        Resource Version:    224913
        UID:                 5152af6a-d538-4bf5-8e90-d48a5b4cf8fe
      Spec:
        Gateway Class Name:  nginx
        Listeners:
          Allowed Routes:
            Namespaces:
              From:  Same
          Name:      http
          Port:      80
          Protocol:  HTTP
          Allowed Routes:
            Namespaces:
              From:  Same
          Name:      https
          Port:      443
          Protocol:  HTTPS
          Tls:
            Certificate Refs:
              Group:
              Kind:       Secret
              Name:       cafe-secret
              Namespace:  certificate
            Mode:         Terminate
      Status:
        Addresses:
          Type:   IPAddress
          Value:  10.1.10.100
        Conditions:
          Last Transition Time:  2024-07-16T18:18:41Z
          Message:               Gateway is accepted
          Observed Generation:   1
          Reason:                Accepted
          Status:                True
          Type:                  Accepted
          Last Transition Time:  2024-07-16T18:18:41Z
          Message:               Gateway is programmed
          Observed Generation:   1
          Reason:                Programmed
          Status:                True
          Type:                  Programmed
        Listeners:
          Attached Routes:  0
          Conditions:
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               Listener is accepted
            Observed Generation:   1
            Reason:                Accepted
            Status:                True
            Type:                  Accepted
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               Listener is programmed
            Observed Generation:   1
            Reason:                Programmed
            Status:                True
            Type:                  Programmed
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               All references are resolved
            Observed Generation:   1
            Reason:                ResolvedRefs
            Status:                True
            Type:                  ResolvedRefs
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               No conflicts
            Observed Generation:   1
            Reason:                NoConflicts
            Status:                False
            Type:                  Conflicted
          Name:                    http
          Supported Kinds:
            Group:          gateway.networking.k8s.io
            Kind:           HTTPRoute
          Attached Routes:  0
          Conditions:
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               Listener is accepted
            Observed Generation:   1
            Reason:                Accepted
            Status:                True
            Type:                  Accepted
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               Listener is programmed
            Observed Generation:   1
            Reason:                Programmed
            Status:                True
            Type:                  Programmed
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               All references are resolved
            Observed Generation:   1
            Reason:                ResolvedRefs
            Status:                True
            Type:                  ResolvedRefs
            Last Transition Time:  2024-07-16T18:18:41Z
            Message:               No conflicts
            Observed Generation:   1
            Reason:                NoConflicts
            Status:                False
            Type:                  Conflicted
          Name:                    https
          Supported Kinds:
            Group:  gateway.networking.k8s.io
            Kind:   HTTPRoute
      Events:       <none>
      ```

    </details>

4. Create the **ReferenceGrant** which permits the **Gateway** in the **default** namespace to
   access the **cafe-secret** Secret in the **certificate** namespace.

    Copy and paste the following code snippet to create the **ReferenceGrant**.

    > If you prefer to manually create and deploy this, click [here](reference-grant.yaml) for the YAML file.

    ```bash
    kubectl create -f - <<EOF
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

    This **ReferenceGrant** allows the Gateway in the `default` namespace to reference the `cafe-secret` Secret in
    the `certificate` Namespace.

    >**Note**: If leave the secret name field blank then the **salve-gateway** will have access to all secrets in the certificate namespace.

    Check the new **referenceGrant** health.

    ```bash
    kubectl -n certificate get referencegrants access-cert-secret
    kubectl -n certificate describe referencegrants access-cert-secret
    ```

    <details>
      <summary><b>Example output</b></summary>

      ```bash
      f5admin@bastion:~$ kubectl -n certificate get referencegrants
      NAME                 AGE
      access-cert-secret   18m
      f5admin@bastion:~$ kubectl -n certificate describe referencegrants access-cert-secret
      Name:         access-cert-secret
      Namespace:    certificate
      Labels:       <none>
      Annotations:  <none>
      API Version:  gateway.networking.k8s.io/v1beta1
      Kind:         ReferenceGrant
      Metadata:
        Creation Timestamp:  2024-07-16T17:57:03Z
        Generation:          1
        Resource Version:    221530
        UID:                 477639ff-a578-4fb7-a055-f91e206a30e8
      Spec:
        From:
          Group:      gateway.networking.k8s.io
          Kind:       Gateway
          Namespace:  default
        To:
          Group:
          Kind:   Secret
          Name:   cafe-secret
      Events:     <none>
      ```

    </details>

5. Create the HTTPRoute resources that does the following
    - Handle HTTPS traffic to `/lanolin`
    - Handle HTTPS traffic to `/arnica`
    - Redirecting HTTP traffic to HTTPS

    However, to make this lab just slightly challenging the following code snippet will create a file
    called **salve-httpRoutes.yaml** but it will not work until you update the values surrounded in
    **##** symbols.

    Refer to [`HTTPRequestRedirectFilter`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.HTTPRequestRedirectFilter) as a reference to `requestRedirect`

    ```bash
    echo "apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: salvadores-tls-redirect
    spec:
      parentRefs:
      - name: salve-gateway
        sectionName: ##What listener name##
      hostnames:
      - "salve.lab.f5npi.net"
      rules:
      - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            port: ##Where to redirect###
    ---
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: lanolin
    spec:
      parentRefs:
      - name: salve-gateway
        sectionName: ##What listener name##
      hostnames:
      - "salve.lab.f5npi.net"
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /lanolin
        backendRefs:
        - name: ##Backend Name##
          port: ##Backend Port##
    ---
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: arnica
    spec:
      parentRefs:
      - name: salve-gateway
        sectionName: ##What listener name##
      hostnames:
      - "salve.lab.f5npi.net"
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /arnica
        backendRefs:
        - name: ##Backend Name##
          port: ##Backend Port##"> salve-httpRoutes.yaml
    ```

    Once you have fixed the YAML configuration file, create the object using the command below.

    ```bash
    kubectl create -f salve-httpRoutes.yaml
    ```

    The section below contains additional details about the **parentRefs** and the solution
    in snippets.

    <details>
      <summary><h3>Expand to reveal parentRefs details</h3></summary>

    To configure HTTPS termination for our **salve** application, we will bind our `lanolin` and `arnica` HTTPRoutes to
    the `https` listener in [salve-httpRoutes.yaml](./salve-httpRoutes.yaml) using
    the [`parentReference`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.ParentReference)
    field:

    ```yaml
    parentRefs:
    - name: salve-gateway
      sectionName: https
    ```

    To configure an HTTPS redirect from port 80 to 443, we will bind the special `salvadores-tls-redirect` HTTPRoute with
    a [`HTTPRequestRedirectFilter`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.HTTPRequestRedirectFilter)
    to the `http` listener:

    ```yaml
    parentRefs:
    - name: salve-gateway
      sectionName: http
    ```

     To configure the backend servers (upstream server or PODs) add the service name and port using the [`backendRef`](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.BackendRef) to the HTTPRoute for both **lanolin** and **arnica**.

    ```yaml
    backendRefs:
    - name: lanolin
      port: 80
    ```

    </details>

## Test the Application

We can now test our use case to validate the following behavior.

1. Confirm HTTPS traffic to **lanolin** and **arnica**
2. Confirm redirecting HTTP traffic to HTTPS traffic

### Confirm HTTPS Termination to Lanolin

### Test Access to Lanolin and Arnica

The **HTTPRoutes** resource named **lanolin** handles incoming HTTPS traffic with the
`salve.lab.f5npi.net` hostname and the **/lanolin** path. Lets confirm HTTPS termination. Since our certificate is self-signed, we will use curl's `--insecure`
option to turn off certificate verification.

```bash
curl --insecure https://salve.lab.f5npi.net/lanolin
```

<details>
  <summary><b>Example output</b></summary>

  Below is a successful HTTPS request.

  ```bash
  f5admin@bastion:~$ curl --insecure https://salve.lab.f5npi.net/lanolin
  Server address: 10.244.67.157:8080
  Server name: lanolin-545d67bfb-4s26l
  Date: 18/Jul/2024:18:03:40 +0000
  URI: /lanolin
  Request ID: a7fe3f4d38b2425154f235c405f4b070
  ```

</details>

### Confirm HTTPS Termination to Arnica

To get arnica:

```bash
curl --insecure https://salve.lab.f5npi.net/arnica
```

The **HTTPRoutes** resource named **arnica** handles incoming HTTPS traffic with the
`salve.lab.f5npi.net` hostname and the **/arnica** path. The server should have a different address. when comparing to the response from **Lanolin**. Lets confirm HTTPS termination.

<details>
  <summary><b>Example output</b></summary>

  Below is a successful HTTPS request.

  ```bash
  f5admin@bastion:~$ curl --insecure https://salve.lab.f5npi.net/arnica
  Server address: 10.244.82.162:8080
  Server name: arnica-6c54c79b6b-sv24j
  Date: 18/Jul/2024:18:04:09 +0000
  URI: /arnica
  Request ID: 1960174fc595c44c389a2d93cee9dbbe
  ```

</details>

### Test HTTP to HTTPS Redirect

The **HTTPRoutes** resource named **salvadores-tls-redirect** handles the HTTP to HTTPS redirects.
NGF will respond with an HTTP 302 Redirect for incoming HTTP request with the
`ferment.lab.f5npi.net` hostname.

Lets give it a try by using `curl` with the `--head` option to only return headers on the Lanolin application.

To get a redirect for lanolin:

```bash
curl --head http://salve.lab.f5npi.net/lanolin
```

<details>
  <summary><b>Example output</b></summary>

  ```bash
  f5admin@bastion:~$ curl --head http://salve.lab.f5npi.net/lanolin
  HTTP/1.1 302 Moved Temporarily
  Server: nginx/1.27.0
  Date: Thu, 18 Jul 2024 18:14:48 GMT
  Content-Type: text/html
  Content-Length: 145
  Connection: keep-alive
  Location: https://salve.lab.f5npi.net/lanolin
  ```

  We can see the **Location** header that has the HTTPS protocol. Notice that the URL uses
  **https** instead of **http**. The resulting HTTPS requests, in this case, is then handled by either the
  **lanolin** HTTPRoute.

</details>

Now, lets add the `--location` switch to the `curl` command to follow the redirect. This results
in a TLS request to **<https://salve.lab.f5npi.net/lanolin>**.

```bash
curl --insecure --head --location http://salve.lab.f5npi.net/lanolin
```

<details>
  <summary><b>Example output</b></summary>

  Here we see the headers from first request in HTTP and the headers from the followup request
  in HTTPS.

  ```bash
  f5admin@bastion:~$ curl --insecure --head --location http://salve.lab.f5npi.net/lanolin
  HTTP/1.1 302 Moved Temporarily
  Server: nginx/1.27.0
  Date: Thu, 18 Jul 2024 18:16:51 GMT
  Content-Type: text/html
  Content-Length: 145
  Connection: keep-alive
  Location: https://salve.lab.f5npi.net/lanolin

  HTTP/2 200
  server: nginx/1.27.0
  date: Thu, 18 Jul 2024 18:16:51 GMT
  content-type: text/plain
  content-length: 164
  expires: Thu, 18 Jul 2024 18:16:50 GMT
  cache-control: no-cache
  ```

</details>

And finally, we can see the response by omitting the `--head` option from the `curl` command.

```bash
curl --insecure --location http://salve.lab.f5npi.net/lanolin
```

<details>
  <summary><b>Example output</b></summary>

  We now see the same response as the HTTPS request.

  ```bash
  f5admin@bastion:~$ curl --insecure --location http://salve.lab.f5npi.net/lanolin
  Server address: 10.244.67.157:8080
  Server name: lanolin-545d67bfb-4s26l
  Date: 18/Jul/2024:18:19:27 +0000
  URI: /lanolin
  Request ID: 58f97e591245d6818c1cdc154ee0cde7
  ```

</details>

You can also repeat this exercise using the will see similar response using the **arnica** route.

```bash
curl --head http://salve.lab.f5npi.net/arnica
curl --insecure --head --location http://salve.lab.f5npi.net/arnica
curl --insecure --location http://salve.lab.f5npi.net/arnica
```

<details>
  <summary><b>Example output</b></summary>

  We now see the same response as the HTTPS request.

  ```bash
  f5admin@bastion:~$ curl --head http://salve.lab.f5npi.net/arnica
  HTTP/1.1 302 Moved Temporarily
  Server: nginx/1.27.0
  Date: Thu, 18 Jul 2024 18:22:46 GMT
  Content-Type: text/html
  Content-Length: 145
  Connection: keep-alive
  Location: https://salve.lab.f5npi.net/arnica

  f5admin@bastion:~$ curl --insecure --head --location http://salve.lab.f5npi.net/arnica
  HTTP/1.1 302 Moved Temporarily
  Server: nginx/1.27.0
  Date: Thu, 18 Jul 2024 18:23:01 GMT
  Content-Type: text/html
  Content-Length: 145
  Connection: keep-alive
  Location: https://salve.lab.f5npi.net/arnica

  HTTP/2 200
  server: nginx/1.27.0
  date: Thu, 18 Jul 2024 18:23:01 GMT
  content-type: text/plain
  content-length: 163
  expires: Thu, 18 Jul 2024 18:23:00 GMT
  cache-control: no-cache

  f5admin@bastion:~$ curl --insecure --location http://salve.lab.f5npi.net/arnica
  Server address: 10.244.82.162:8080
  Server name: arnica-6c54c79b6b-sv24j
  Date: 18/Jul/2024:18:23:09 +0000
  URI: /arnica
  Request ID: 36d322bb0d2cae4192311711b0105553
  ```

</details>

You can find the solution for the Salve HTTPRoutes below.

<details>
  <summary><h3><b>Solution</b><h3></summary>

  [Click here](salve-httpRoutes.yaml) to see a solution for the <b>HTTPRoute</b> configurations.
</details>

>**Question**: What parts of the k8s configuration creates these redirects? How can we change them?

<details>
  <summary><b>Answer<b></summary>

  The HTTPRoute is the kubernetes resource that contain the configuration for these redirects.
  In the case here, it is called **salvadores-tls-redirect** and uses the **RequestRedirect** filter. Below is a snippet of the YAML file showing the redirect.

  ```yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: salvadores-tls-redirect
  spec:
    rules:
    - filters:
      - type: RequestRedirect
        requestRedirect:
          scheme: https
          port: 443
  ```

</details>

>**Question**: What parts of the k8s configuration routes incoming requests the backend **arnica** and **lanolin** Pods? How can we tell the names of the pods that they redirect to?

<details>
  <summary><b>Answer<b></summary>

  The HTTPRoute is also the kubernetes resource that configures the endpoint of a route. So in this
  case, there are two HTTPRoutes configured, one for a route to **arnica** and another to
  **lanolin**

  We will look at a snippet of a YAML output from the **arnica** HTTPRoute as an example here.
  This route references a Service named **arnica** on port **80**.

  The command to output YAML format of the **arnica** HTTPRoute resource.

  ```bash
  kubectl get httproutes arnica  -oyaml
  ```

  Output Snippet

  ```yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: arnica
    namespace: default
  spec:
    rules:
    - backendRefs:
      - name: arnica
        kind: Service
        port: 80
  ```

  Now that we know the **arnica** HTTPRoute is referencing the **arnica** Service, the next step is
  to find out the **arnica** Service to Pod association. In the example here, the configuration
  that defines the pod association is in the **selector** portion of the YAML output. So in this
  case, it is `app: arnica`.

  ```bash
  kubectl get services arnica -oyaml
  ```

  Output Snippet

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: arnica
    namespace: default
  spec:
    selector:
      app: arnica
  ```

Finally, we can look at the labels of the pods. Notice the **arnica-6c54c79b6b-sv24j** Pod have the
label `app=arnica`. So the **arnica** Service is associated to the **arnica-6c54c79b6b-sv24j** Pod.

```bash
kubectl get pods --show-labels
```

  Output

  ```bash
  f5admin@bastion:~$ kubectl get pods --show-labels
  NAME                      READY   STATUS    RESTARTS   AGE   LABELS
  arnica-6c54c79b6b-sv24j   1/1     Running   0          41m   app=arnica,pod-template-hash=6c54c79b6b
  lanolin-545d67bfb-4s26l   1/1     Running   0          41m   app=lanolin,pod-template-hash=545d67bfb
  ```

Now confident the HTTPRoute **arnica** is routed to the **arnica** application where the pod is
named **arnica-6c54c79b6b-sv24j**.

</details>

## Clean up

When done with this lab you can clean up the objects by running the following commands.

```bash
kubectl delete deployments arnica lanolin
kubectl delete services arnica lanolin
kubectl delete namespaces certificate
kubectl delete gateways salve-gateway
kubectl delete httproutes salvadores-tls-redirect lanolin arnica
```

## Source files

Here are the configuration files in the event that you want to manually go through this lab.

- [Salvador's Salve Shop application](salve.yaml)
- [Namespace and Certificate](certificate-ns-and-cafe-secret.yaml)
- [ReferenceGrant for certificate namespace](reference-grant.yaml)
- [The Gateway resource](salve-gateway.yaml)

## Solution file

- [Working HTTPRoute for the Salvador's Salve Shop](salve-httpRoutes.yaml)

Previous: [Demo](../demo/README.md)

Next: [Fix It Lab](../fixit/README.md)

---

## End of lab
