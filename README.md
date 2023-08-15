# LDAP-with-Dexdddd
Implentation of ldap with dex in kubernetes cluster
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

