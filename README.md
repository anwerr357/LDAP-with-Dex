# LDAP-with-Dexdddd
Implentation of ldap with dex in kubernetes cluster
Note: this repository prerequests that the kubernetes cluster is already setted in your environement.
![Anwer Lahami](https://www.arrikto.com/wp-content/uploads/2019/07/istio-dex-authentication.png)
## cert-manager
cert-manager is used by many Kubeflow components to provide certificates for admission webhooks.
```
kustomize build common/cert-manager/cert-manager/base | kubectl apply -f -kubectl wait --for=condition=ready pod -l 'app in (cert-manager,webhook)' --timeout=180s -n cert-manager
kustomize build common/cert-manager/kubeflow-issuer/base | kubectl apply -f -
 ```
### Error:
In case you get this error:
```
 Error from server (InternalError): error when creating "STDIN": Internal error occurred: failed calling webhook "webhook.cert-manager.io": failed to call webhook: Post "https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=10s": dial tcp 10.96.202.64:443: connect: connection refused 
```
This is because the webhook is not yet ready to receive request. Wait a couple seconds and retry applying the manfiests.
For more troubleshooting info also check out https://cert-manager.io/docs/troubleshooting/webhook/
##  istio:
We will focus on using Istio to perform end-user authentication, meaning that our apps won’t contain any authentication logic!
Authentication, for user access to an application, will be done at the Istio Gateway: the one point where all traffic enters the cluster.

Authentication is a major area that developers may choose to leave up to Istio. The idea is simple:
![Anwer Lahami](https://www.arrikto.com/wp-content/uploads/2019/07/Istio_Gateway_Overview-1024x755.jpg)
Incoming traffic includes a JSON Web Token (JWT) for authentication.
The JWT is verified by the Istio Gateway.
### install istio:
```
kustomize build common/istio-1-16/istio-crds/base | kubectl apply -f -
kustomize build common/istio-1-16/istio-namespace/base | kubectl apply -f -
kustomize build common/istio-1-16/istio-install/base | kubectl apply -f -
```
## Dex:

Apps inside the cluster trust the JWT because it has been verified by the Gateway.
