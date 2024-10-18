# Upgrade NGINX Gateway Fabric

https://docs.nginx.com/nginx-gateway-fabric/installation/ngf-images/jwt-token-docker-secret/
Note: This is not a production lab training course, so we are not going to do a hitless rolling deployment, nor canary, nor blue green.  Just a straight upgrade.

1. get your JWT from your myF5
2. copy your JWT to your /home/f5demo directory on the bastion host (maybe by copy and pasting (shift-insert, or ctrl-shift-v) it into a file )
3. docker is installed, so do a docker login: docker login private-registry.nginx.com --username=`cat myF5.jwt` --password=none
5. create a secret: kubectl create secret docker-registry nginx-plus-registry-secret --docker-server=private-registry.nginx.com --docker-username=`cat myF5.jwt` --docker-password=none -n nginx-gateway
6. check it out: kubectl get secret nginx-plus-registry-secret --output=yaml
8. deploy the latest NGF
- kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.4.0" | kubectl apply -f -
- kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/crds.yaml
- kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/nginx-plus/deploy.yaml
- kubectl get pods -n nginx-gateway
