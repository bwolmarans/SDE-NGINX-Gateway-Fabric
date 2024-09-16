# Cross Namespace Routing Use Case

We will go though configuring and validating a use case where traffic is routed across different
namespaces.

## Prerequisites

* An [SDE NGINX Gateway Fabric UDF Blueprint](https://udf.f5.com/b/d2617e7e-018f-4c9a-a15f-09ca55ae8a37) deployment

## Overview

The NGF solutions allows Application Administrators to configure traffic such that the incoming
route configuration is in a different namespace than the application.

## Up-skill session

We will now go into the actual labs which will consist of the following:

1. [Hats Demo](demo/README.md): Configure cross namespace routing to a *hats* application.
2. [Shoes Lab](lab/README.md): You will configure cross namespace routing to a *shoes* application.
3. [Coats Fix It Lab](fixit/README.md): There is a *coats* application with broken configurations,
   find and fix the issue.

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)
