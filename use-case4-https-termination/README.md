# HTTPS Termination Use Case

We will go though configuring and validating traffic with HTTPS Termination.

## Prerequisites

* An [SDE NGINX Gateway Fabric UDF Blueprint](http://go/sde-ngf-udf) deployment

## Overview

NGINX Gateway Fabric supports client side HTTPS termination.  Support for server side HTTPs termination is on the roadmap but not part of the **stable** release at this time. NGINX Gateway Fabric also supports HTTP to HTTPS redirection.

## Up-skill session

We will now go into the actual labs which will consist of the following:

1. [Fermentation Demo](demo/README.md): Configure HTTPS Termination to a *Fermentation* application.
2. [Transaction Lab](lab/README.md): You will configure HTTPS Termination to a *transaction*
   application.
3. [Fix It Lab](fixit/README.md): Find and fix the issue.

## Additional resources

For more information on the Kubernetes HTTPRoute Filter and the RequestHeaderModifier, consider
the following resources:

* [HTTP to HTTPS redirects](https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/?h=)
* [TLS Termination](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/?h=termination#tls-termination)

Previous: [NGINX Gateway Fabric](../README.md)

Next: [Demo](demo/README.md)
