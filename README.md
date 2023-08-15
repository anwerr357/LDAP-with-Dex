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
Apps inside the clIstio Gateway enforces Auth for the Kubeflow apps

This way, our apps contain no authentication logic at all!
Unfortunately, it’s not that simple.uster trust the JWT because it has been verified by the Gateway.
Authentication in Istio
At first glance, Istio seems to support end-user authentication.
However, notice how Istio can only perform the last part, token verification (i.e. verify the JWT and allow the request).
The question is: how are we going to get that token in the first place?

Enter OpenID Connect (OIDC): a way to authenticate a user using a standardized OAuth2 flow.

The OIDC Flow
Because a picture is worth a thousand words, let’s take a look at what an OIDC flow looks like.
![Anwer Lahami](https://www.arrikto.com/wp-content/uploads/2020/09/OIDC_Flow-.png)
The OIDC Flow — Istio Gateway only supports JWT verification
Notice how Istio can only perform the last part, token verification.
Because of this, we need a new entity that will act as the OIDC client and execute the flow.

## Dex:
Dex is an OpenID Connect Identity (OIDC) with multiple authentication backends. In this default installation, it includes a static user with email user@example.com. By default, the user's password is 12341234. For any production Kubeflow deployment, you should change the default password by following the relevant section.

Install Dex:
```
kustomize build common/dex/overlays/istio | kubectl apply -f -
```
## OIDC AuthService:
The OIDC AuthService extends your Istio Ingress-Gateway capabilities, to be able to function as an OIDC client:
```
kustomize build common/oidc-authservice/base | kubectl apply -f -
```
till now that's all what you need to build you project !!!
## Putting it all Together:
Our Istio Gateway can now act as an OIDC client and execute the whole flow to authenticate a user.
Let’s test it out using Dex, a popular OIDC provider.
Dex supports many authentication backends, including static users, LDAP and external Identity Providers, so you can have the power of choice.

The final architecture is the following:
![Anwer Lahami](https://www.arrikto.com/wp-content/uploads/2020/09/Istio-Gateway-auth.jpg)

## Demo:
Access the dex-config.yaml file, and you'll observe the absence of connectors 
```
kubectl get configmap dex -n auth -o jsonpath='{.data.config\.yaml}'
```
linked to the LDAP server intended for authentication. To rectify this, we will modify the file according to the provided instructions:
```
cat << EOF >> dex-config.yaml
connectors:
- type: ldap
  id: ldap
  name: LDAP
  config:
    host: DC1.vdi.sclabs.net
        #This is the user which has read access to AD
    bindDN: "cn=kates,ou=UEMUsers,dc=vdi,dc=sclabs,dc=net"
        #This is the password for the above account
    bindPW: <pwOfKatesUser>
        #What the user is going to see in Kubeflow
    usernamePrompt: "vdi user + domain, e.g. 'kevin@vdi.sclabs.net'"
    userSearch:
          #Which AD/LDAP users may access Kubeflow
      baseDN: ou=UEMUsers,dc=vdi,dc=sclabs,dc=net
          #This is the mapping I've talked about and I'll explain again
      username: userPrincipalName
      idAttr: sAMAccountName
      emailAttr: userPrincipalName
      nameAttr: displayName
EOF
```
### merge the new config file with the  with actual: 
```
kubectl create configmap dex \
--from-file=config.yaml=dex-config.yaml \
-n auth --dry-run -oyaml | kubectl apply -f -
```
###reapply auth
```
kubectl rollout restart deployment dex -n auth
```
You should be greeted by the Dex login screen
![Anwer Lahami](https://miro.medium.com/max/605/1*glFuBc_JrV063KqGnclvYA.png)
