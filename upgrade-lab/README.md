# Upgrade NGINX Gateway Fabric

Note: This is not a production environment, so we are not going to do a hitless rolling deployment, nor canary, nor blue green.

[Kubernetes Rolling Update Information](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)



### [Release Notes for NGF](https://github.com/nginxinc/nginx-gateway-fabric/blob/main/CHANGELOG.md)


1. check the version before upgrade:
```bash
k -n nginx-gateway describe gatewayclasses.gateway.networking.k8s.io nginx 
```
OUTPUT:
```bash
Name:         nginx
Namespace:
Labels:       app.kubernetes.io/instance=nginx-gateway
              app.kubernetes.io/name=nginx-gateway
              app.kubernetes.io/version=1.3.0
```

2. Get your JWT from one of your NGINX Subscriptions in MyF5
3. copy your JWT to your /home/f5demo directory on the bastion host (maybe by copy and pasting (shift-insert, or ctrl-shift-v) it into a file )
4. docker is installed, so do a docker login: 

```bash
docker login private-registry.nginx.com --username=`cat myF5.jwt` --password=none
```

5. create a secret:
```bash
kubectl create secret docker-registry nginx-plus-registry-secret --docker-server=private-registry.nginx.com --docker-username=`cat myF5.jwt` --docker-password=none -n nginx-gateway
```
6. confirm the secret exists:
   
```bash
kubectl get secret nginx-plus-registry-secret -n nginx-gateway --output=yaml
```
7. deploy the latest NGF
```bash
kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.4.0" | kubectl apply -f -
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/nginx-plus/deploy.yaml
kubectl get pods -n nginx-gateway
```

8. check the version after upgrade:
```bash
k -n nginx-gateway describe gatewayclasses.gateway.networking.k8s.io nginx
```
OUTPUT:
```bash
Name:         nginx
Namespace:
Labels:       app.kubernetes.io/instance=nginx-gateway
              app.kubernetes.io/name=nginx-gateway
              app.kubernetes.io/version=1.4.0
```
