# Modify HTTP Request Headers Use Case

Welcome to this repository where we explore and demonstrate the utilization of the
RequestHeaderModifier filter within HTTPRoute, a feature provided by the Kubernetes Gateway API.
This README serves as a guide to understanding how we can leverage this feature to modify HTTP
request headers dynamically.

## Prerequisites

* An [SDE NGINX Gateway Fabric UDF Blueprint](https://udf.f5.com/b/d2617e7e-018f-4c9a-a15f-09ca55ae8a37) deployment

## Overview

The RequestHeaderModifier filter allows us to alter the headers of HTTP requests as they pass
through a Gateway. This capability is particularly useful in scenarios where you want to inject,
remove, or alter request headers for traffic management, security enhancements, or to enrich the
requests with additional information.

In this use-case breakout, we will deploy NGINX Gateway Fabric and set up traffic routing to a simple echo server. The echo server will serve as a backend service, allowing us to observe the changes made to the request headers through the use of HTTPRoute resources and the RequestHeaderModifier filter.

## Up-skill session

We will now do a dive deeper into each aspect of this use-case which will consist of the following:

1. [Modify Request Headers Demo](demo/README.md): Follow step-by-step set up the demonstration.
2. [Modify Request Headers Lab](lab/README.md): Break into smaller groups and configure NGF to modify request headers.
3. [Remove Request Headers Fix It Lab](fixit/README.md):
<!-- * [Troubleshooting README](troubleshooting/README.md): Find data gathering and debugging techniques. -->

## Additional resources

For more information on the Kubernetes HTTPRoute Filter and the RequestHeaderModifier, consider
the following resources:

* [Kubernetes HTTPRouteFilter](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.HTTPRouteFilter)
* [Kubernetes RequestHeaderModifier](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.HTTPHeaderFilter)

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)
