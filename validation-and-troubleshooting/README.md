# Validation and troubleshooting

This is a collection of validation steps along with troubleshooting tips that can be used to confirm an environment is healthy.  This includes the underlying Kubernetes cluster and also the NGINX Gateway Fabric instance.

## Prerequisites

* A running instance of the [SDE NGINX Gateway Fabric UDF Blueprint](https://udf.f5.com/b/d2617e7e-018f-4c9a-a15f-09ca55ae8a37) deployment.

### shell updates to improve your Kubernetes interactions

There are some common commands that you might want to set as shell variables.

1. Configure the option to sort output by creation time as a variable.  Here we are creating a variable called **sb** (sort-by) to --sort-by=.metadata.creationTimestamp

```bash
export sb='--sort-by=.metadata.creationTimestamp'
```

2. When creating kubernetes objects using the imperative redirect the output to a yaml file for validation and possible updates.

```bash
export do='--dry-run=client -o yaml'
```

3. Create a variable that will verify your yaml file without actually creating the objects in Kubernetes.

```bash
export dos='--dry-run=server'
```
1. Create a .vimrc file that at least sets some common yaml editor values.  You might consider adding highlighting for the current column and line too `set cuc cul`.

```bash
color desert
set tabstop=2
set expandtab
set shiftwidth=2
set cuc cul" > ~/.vimrc
```

### Checking the Kubernetes cluster health

If you are initially checking a new Kubernetes cluster you might run through the following checks to validate the cluster health.

Check the nodes, ensure they have a status of **Ready**.

```bash
kubectl get nodes -o wide
```

If any of the nodes show something other than Ready you might review the latest details using the describe option

```bash
kubectl describe nodes w1-mgmt.lab.f5npi.net
```

Check all the pods

```bash
kubectl get pods --all-namespaces
```
Check for any pod that is explicitly not in the the running state.

```bash
kubectl get pods --all-namespaces --field-selector=status.phase!=Running
```

Next you might review the logs for some of the core Kubernetes services.  You can often accomplish this using journalctl for the kubelet process and sometimes other processes.

>**Note**: See a list of available systemd units using `journalctl --field _SYSTEMD_UNIT`

```bash
journalctl -u kubelet
```

Alternatively you can also check the logs for most to the kubernetes core services using the log function and the pod name. The coreDNS and Kube-proxy pods are deployment sets so the pod names are dynamic.  As such you will have to enter their names explicitly to check the logs.

```bash
kubectl -n kube-system logs pods/kube-apiserver-ctlr-mgmt.lab.f5npi.net
```

```bash
kubectl -n kube-system logs pods/kube-scheduler-ctlr-mgmt.lab.f5npi.net
```

```bash
kubectl -n kube-system logs pods/kube-controller-manager-ctlr-mgmt.lab.f5npi.net
```

```bash
kubectl -n kube-system logs pods/kube-proxy-[TAB][TAB]
```

Check the kubernetes documentation [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/) for additional details on this topic.

### Checking the NGINX Gateway Fabric health

We can check all the common services and pods using `get all`.  Here we want to make sure the service is available and confirm how it is exposed externally via a NodePort or LoadBalancer IP address.  We also want to be sure the POD is running healthy and shows two containers running.

```bash
kubectl -n nginx-gateway get all
```

## --- STOP HERE - The rest is for Troubleshooting only if needed later in the lab ---

Next we might look for more details related to the nginx-gateway pod using the **describe** option.

```bash
kubectl -n nginx-gateway describe pod nginx-gateway-[TAB]
```

>**Note**: You can query the pod for container names using the deployment like this:
>`kubectl -n nginx-gateway get deployments nginx-gateway -o jsonpath='{.spec.template.spec.containers[*].name}'​`

Next you might want to review the logs for both of the nginx-gateway fabric containers.

```bash
kubectl -n nginx-gateway logs pods/nginx-gateway-[TAB] nginx-gateway
```
```bash
kubectl -n nginx-gateway logs pods/nginx-gateway-[TAB] nginx
```

### NGINX Gateway Fabric Gateway and Gatewayclass health

The GatewayClass and Gateway objects are the building blocks for application routes in NGINX Gateway Fabric.  The Gateway Class options are installed at the cluster level and often by the Infrastructure Providers.  One of the Gateway Class Controllers is of course the NGINX Gateway Fabric.

Check for all available Gateway Classes.  We want to confirm that the NGINX Gateway Controller has an **ACCEPTED** state of **True**. We can get additional details about the GatewayClass using the `-o yaml` option with `kubectl get` or using the describe option.  When using the describe option review the Events to confirm there are not events of concern.

```bash
kubectl get gatewayclasses
```
```bash
kubectl get gatewayclasses nginx -o yaml
```
```bash
kubectl describe gatewayclasses nginx
```

Next review the Gateway objects that reference your preferred GatewayClass.  The Gateway PROGRAMMED status should be True.  In addition if your NGINX Gateway Fabric is exposed used a load balancer service you should see that IP address listed here too.

>**Note**: In these examples we are working with a gateway named **cafe-gateway**.

```bash
kubectl get gateways cafe-gateway
```

Just as before we can get more details about our Gateway using `-o yaml` and the describe function.  When using the describe function check the Events for any issues of concern.

```bash
kubectl get gateways cafe-gateway -o yaml
```
```bash
kubectl describe gateway cafe-gateway
```

### NGINX Gateway Fabric HTTPRoute health

The HTTProute objects rely on the GatewayClass and Gateway objects.

First we can check for any HTTPRoute in the default namespace and then in any namespace.

```bash
kubectl get httproutes
```
```bash
kubectl get httproutes --all-namespaces
```

>**Note**: In this example we are reviewing the **coffee-httproute** object.

We can dig a little further and look for the HTTPRoute rules, backend references and more using the `-o yaml` option or the describe function.

```bash
kubectl get httproutes coffee-httproute  -o yaml
```
```bash
kubectl describe httproutes coffee-httproute
```

You can also review the NGINX configuration that was pushed to the NGINX container by the NGINX gateway controller. Any rules created, matches defines, hostnames listed etc should be configured in the NGINX configuration also.  If not then it is very likely that something was misconfigured in the NGINX Gateway Fabric objects.

```bash
kubectl -n nginx-gateway exec deployments/nginx-gateway -c nginx -- nginx -T
```

### NGINX Gateway Fabric Custom Resources

The GatewayClass, Gateway, HTTPRoute and ReferenceGrants all have relationships between them.

You can verify these using `kubectl get` and the `-o yaml` option.  For example the Gateway object should refer to the GatewayClass of type nginx in these labs.

```bash
kubectl get gatewayclasses nginx -o yaml
```
```bash
kubectl get gateways cafe-gateway -o yaml
```

The HTTPRoute object should refer to the Gateway object and the listener section defined.

```bash
kubectl get gateways cafe-gateway -o yaml
```
```bash
kubectl get httproutes coffee-httproute -o yaml
```

___
___
