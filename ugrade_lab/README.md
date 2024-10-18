# Upgrade NGINX Gateway Fabric

## Note: This is not a production environment, so we are not going to do a hitless rolling deployment, nor canary, nor blue green.

1. check the version before upgrade: k -n nginx-gateway describe gatewayclasses.gateway.networking.k8s.io nginx 
2. copy your JWT to your /home/f5demo directory on the bastion host (maybe by copy and pasting (shift-insert, or ctrl-shift-v) it into a file )
3. docker is installed, so do a docker login: docker login private-registry.nginx.com --username=\`cat myF5.jwt\` --password=none
5. create a secret: kubectl create secret docker-registry nginx-plus-registry-secret --docker-server=private-registry.nginx.com --docker-username=\`cat myF5.jwt\` --docker-password=none -n nginx-gateway
6. check it out: kubectl get secret nginx-plus-registry-secret --output=yaml
8. deploy the latest NGF
- kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.4.0" | kubectl apply -f -
- kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/crds.yaml
- kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/nginx-plus/deploy.yaml
- kubectl get pods -n nginx-gateway
9. check the version after upgrade: k -n nginx-gateway describe gatewayclasses.gateway.networking.k8s.io nginx 

## Let's also enable the NGINX Plus Dashboard

In a new webshell:

- Port Forward to see Plus API: kubectl port-forward -n nginx-gateway nginx-gateway-7b97d65fc7-mc8jj --address 0.0.0.0 8765:8765 -n nginx-gateway &

- Generate Traffic: for i in {1..1000}; do curl tea.lab.f5npi.net/tea ; done

- Now connect in UDF using Bastion Host GUI access method in a new browser tab


