# Split Traffic Connections Use Case

We will go though configuring and validating a use case where traffic connections are split.

## Prerequisites

* An [SDE NGINX Gateway Fabric UDF Blueprint](https://udf.f5.com/b/d2617e7e-018f-4c9a-a15f-09ca55ae8a37) deployment

## Overview

This unit is focused on how to route traffic using weighting.  The backend services have a default weight value of 1 but this can changed to to any value.  The the weights are proportional and do not have to be based on a percent of 100.   Setting the weigh to 0 disables all traffic for the PODs in that service.​

## Up-skill session

We will now go into the actual labs which will consist of the following:

1. [Coffee Demo](demo/README.md): Configure splitting traffic connections to different *coffee*
   applications.
2. [Tea Lab](lab/README.md): You will configure splitting traffic connections to different
   *tea* applications.
3. [Coats Fix It Lab](fixit/README.md): There is a *coats* application with broken configurations,
   find and fix the issue.

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)
