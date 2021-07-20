# Introduction 
This repo contains traefik ingress helm chart configuration for flow-v2 cluster.

## Docs
https://github.com/traefik/traefik-helm-chart

**_NB:_** this guide uses dev env as an example - please, **change the Resource Group and Azure Kubernetes Cluster names according to the environment you want to modify, as well as certificate secret name** (_%env_name%-api-cert_). 

Most of the configuration is done via [this guide](https://docs.traefik.io/getting-started/install-traefik/#use-the-helm-chart).
Prerequisites:
- bash/zsh/other POSIX-compatible shell
- azure CLI authorized for resource group where your cluster located
- existing AKS cluster
- helm v3.x

To install traefik as ingress provider you need:
1. Configure kubectl to use your AKS cluster 
```
RG=tg-dev
AKS=eastus2-dev-tg-cluster
INGRESS_NAMESPACE=traefik-ingress

az aks get-credentials --resource-group $RG --name $AKS
```

2. Obtain SSL certificate with private key (now it's done through IT department).

Please, check files encoding before usage - they should be in UTF-8 (**without** BOM); otherwise, they won't be parsed properly. Name the certificate files as _tls.crt_ and key as _tls.key_, respectively.

3. Create the Kubernetes secret to store the certificate.
```
kubectl create ns $INGRESS_NAMESPACE
kubectl create secret generic dev-api-cert --from-file=tls.crt=src/dev/fullchain.crt --from-file=tls.key=src/dev/privkey.key -n $INGRESS_NAMESPACE
```
Here we use _dev-api-cert_ name. Note that **the certificate, TLS Store (see p. 6) and Ingress service must be located in the same namespace**, while dedicated routes for specific services should be located in services' namespaces (see https://github.com/kubernetes/kubernetes/issues/17088).

4. Add helm repo
```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```
It's important to use chart from `containous` repo, because chart in `helm` repository is outdated and contain only v1.x traefik images.

5. Install traefik [with configs from our repo src folder](https://dev.azure.com/ASGINC/FlowV2/_git/FlowV2.traefik?path=%2Fsrc):
```
helm install --namespace=$INGRESS_NAMESPACE traefik traefik/traefik 
```

**_Tip:_** if you want to modify your values file (here it's _dev.values.yaml_) and update the config on-the-fly without the need to redeploy the Traefik fully (which implies recreation of the load balancer and the need to modify A record for new IP), just use the `helm upgrade` command with the same parameters as above - this way the load balancer is kept and only Traefik pod(s) are updated. Still, in case of any problems it's still better to reinstall the traefik.

6. Configure the default TLS Store to hold the certificate from p. 3.

```
kubectl apply -f ./src/dev/tls_store.yaml
```
If your certificate secret name differs from _dev-api-cert_ (e.g. for another environment), just update _tls_store.yaml_ to use this name before applying.

**_Tip:_** if you update the store to use another secret or make changes to existing secret specified in the store, there's no need to redeploy existing Traefik pod (if any) - these changes will be monitored automatically.

7. Add HTTP to HTTPS redirection.

```
kubectl apply -f ./src/dev/ingress_https_redirect.yaml
```
If you're performing the setup for hosts other than dev-tg-api.somniumgame.com for APIs and dev-ui.bizprotect.org for UI (e.g. for another environment), just update the _ingress_https_redirect.yaml_ to use it before applying. Then wait for a few seconds. Now http://dev-tg-api.somniumgame.com/ should redirect to https://dev-tg-api.somniumgame.com/, and should provide your certificate to the client (if you chose other domain, please use the one you've specified in configuration).

8. Get load balancer's external IP:
```
LOAD_BALANCER_EXTERNAL_IP=$(kubectl get services -n $INGRESS_NAMESPACE | awk '{print $4}' | tail -n1)
echo $LOAD_BALANCER_EXTERNAL_IP
```
Be sure `LOAD_BALANCER_EXTERNAL_IP` was actually set, it can take some time for AKS to allocate it, so you may receive something like `pending` instead of valid IPv4. If you do - just wait for a few minutes and try get IP again.

9. Ask @<47762F17-2079-6BE7-B6E8-CE69AA9D63DC> for GoDaddy access (if you don't have it yet) and add A record using IP from p. 8 for needed subdomain. It could take some time (from 3 min to 2 hours) for DNS to announce your domain/IP record. If you need to send some requests right on - just write domain/IP pair into your system hosts file or configure it on your network router DNS.

Example (now services are deployed by CI/CD - no need to do this manually!)
-
Check `example/` folder in [our traefik repository](https://dev.azure.com/ASGINC/FlowV2/_git/FlowV2.traefik?path=%2Fexample) if you need help with configuring ingressRoutes or [check related documentation](https://docs.traefik.io/routing/providers/kubernetes-crd/).
Example contains simple nginx deployment with 3 replicas, kubernetes service declaration for it and traefik ingressRoute that will listen all requests and forward it to your nginx pods.
```
kubectl create ns example-traefik-v2
kubectl apply -f ./example/deployment.yaml
kubectl apply -f ./example/service.yaml
kubectl apply -f ./example/ingress_route.yaml
```
As you may have noticed, the route uses only HTTPS port since we've added HTTP to HTTPS redirection before.


