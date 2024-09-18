# Basic Host and Path-based Routing

We will go though configuring and validating a basic use case using host and path-based routing.

## Prerequisites

* An [SDE NGINX Gateway Fabric UDF Blueprint](https://udf.f5.com/b/d2617e7e-018f-4c9a-a15f-09ca55ae8a37) deployment

## Overview

The NGF solutions allows Application administrators to configure inbound rules allowing traffic to
their kubernetes applications using either hostname or uri path values.

This lesson includes one demo, one interactive lab and one break fix lab.

During the host and path routing unit of the up-skilling session there is a short demo highlighting [how **Pods** can be exposed using
services](clusterip-nodeport-loadbalancer.md). This is a link to that PODS to Services demo.

1. [Coffee Demo](demo/README.md): Configure basic routing to a *coffee* application.
2. [Tea Lab](lab/README.md): You will configure basic routing to a *tea* application,
3. [Coffee and Socks Fix It Lab](fixit/README.md): There are *coffee* and *socks* applications with
   broken configurations, find and fix the issue.

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)
