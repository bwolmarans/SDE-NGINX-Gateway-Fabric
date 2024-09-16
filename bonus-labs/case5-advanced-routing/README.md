# Advanced Routing

This repository includes the labs and guides to support the SDE up-skilling session: **Advanced Routing**.


## Prerequisites

* An [SDE NGINX Gateway Fabric UDF Blueprint](https://udf.f5.com/b/d2617e7e-018f-4c9a-a15f-09ca55ae8a37) deployment

## Overview

We have seen examples for routing, modifying headers, HTTPS termination, and others. NGINX Gateway Fabric has the additional capability to route traffic dynamically based on combinations of [Kubernetes HTTPRouteRules](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPRouteRule) from the API Gateway configuration of the cluster. 


This includes the capability to route based on:

- HTTP Hostname
- HTTP Methods: POST, GET
- HTTP Headers by Exact Match and Path Match

This lesson includes one demo, one interactive lab and one break fix lab.

During the up-skilling session there is a short demo highlighting how **Pods** can be exposed using
services. If you would like to run through the same steps go to the
[Services demo](clusterip-nodeport-loadbalancer.md).

## Up-skill session

We will now go into the actual labs which will consist of the following:

1. [Post-n-Get Demo](demo/README.md): Configure basic routing to a *coffee* application.
2. [Hostname Lab](lab/README.md): You will configure basic routing to a *tea* application,
3. [Coffee and Socks Fix It Lab](fixit/README.md): There are *coffee* and *socks* applications with
   broken configurations, find and fix the issue.

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)



----------- OLD OLD OLD ----------------
# Configuring Advanced Routing

This repository includes the labs and guides to support the SDE up-skilling session: **Configuring HTTPS Termination**.

It is expected that you are completing these steps using the SDE NGF UDF lab. You can start that UDF lab now if you have not already done so.

[go/sde-ngf-udf](https://udf.f5.com/b/d2617e7e-018f-4c9a-a15f-09ca55ae8a37)

## Overview

# Short Description of uc

This lesson includes one demo, one interactive lab and one break fix lab.

## Labs

We start with the demo lab.  Users can follow along and run through the same set of commands and configurations in their running UDF lab environments.

- [Demo](demo/README.md)

Next during the up skilling session participants will join break out groups and complete the Tea Lab.  This lab is very close to the Coffee Demo but does require that the participants manually update the HTTPRoute configuration file.

- [Lab](lab/README.md)

The last exercise of the up-skilling session asks the participants to deploy a broken solution and then using their validation and troubleshooting knowledge solve the mystery.

- [Fix](fix/README.md)

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)
