# Advanced Routing and Traffic Splitting

This repository includes the labs and guides to support the SDE up-skilling session **Advanced Routing and Traffic Splitting**. It is designed to be used together with use case 5 and 6 in the [Powerpoint presentation](https://f5.sharepoint.com/:p:/r/sites/gscoe/Shared%20Documents/General/UpSkilling%20content/NGINX/NGINX%20Gateway%20Fabric/Combined%20Use%20Case%20Presentation%20Deck.pptx?d=wee10d3dd72094f47a32609390966e06e&csf=1&web=1&e=teg0ck).

## Prerequisites

* An [SDE NGINX Gateway Fabric UDF Blueprint](http://go/sde-ngf-udf) deployment

## Overview

We have seen examples for cross-namespace routing, modifying headers, HTTPS termination, and others. NGF also has the capability to route traffic dynamically based on combinations of [Kubernetes HTTPRouteRules](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPRouteRule) from the API Gateway configuration of the cluster.

This includes the capability to route or split traffic based on:

* HTTP Hostname
* HTTP Methods: POST, GET
* HTTP Headers by Exact Match and Path Match

Additionally, **weight** can be placed on each ```backendRef```, allowing Services to be statistically favored, disfavored, or disabled.

## Up-skill session

We will now go into labs which will consist of the following:

1. [Demo](demo/README.md): Demo of routing based on HTTP Method, HTTP Header, and weight
2. [Advanced Routing and Traffic Splitting](lab/README.md): Set up traffic splitting in the lab

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)
