# AKS cluster for Faceanalyzer

This repo contains configuration for setting up an AKS cluster to which Faceanalyzer application can be deployed.

## Ingress-nginx

Ingress-nginx is a reverse proxy. It enables external access to ingresses in cluster.

1. Navigate to `ingress-nginx/`.
2. Run `helmfile apply`.
3. Run `kubectl get svc -n ingress-nginx -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'` to get IP adress of ingress-nginx.
4. Add the IP adress to the DNS provider as A record.

## Cert-manager

Cert-manager issues Let's Encrypt certificates for TLS.

1. Navigate to `cert-manager/`.
2. Run `helmfile apply`.

## Namespace

Namespace for Faceanalyzer application should be created manually. Access to cluster is needed.

```bash
az login
az aks get-credentials --resource-group faceanalyzer --name faceanalyzer
kubelogin convert-kubeconfig
kubectl create namespace faceanalyzer
```

## Get kubeconfig as service principal

CI/CD uses kubeconfig to deploy Faceanalyzer application the cluster. This requires service principal on Azure. CI/CD uses `faceanalyzer-cicd` principal. You can find it on Azure under Enterprise applications.

To get the principal's kubeconfig and test if it works, first get your kubeconfig and then convert it:

```bash
az aks get-credentials --resource-group faceanalyzer --name faceanalyzer
export AAD_SERVICE_PRINCIPAL_CLIENT_ID=824f57a1-191c-4ace-ac4e-09ff215a7cfe
export AAD_SERVICE_PRINCIPAL_CLIENT_SECRET=seCreTStrInG
kubelogin convert-kubeconfig -l spn
kubectl get namespaces
```

Then, you need to add the kubeconfig to our CI/CD service, GitHub Actions.

1. Open backend repo and go to Settings > Secrets and Variables > Actions.
2. Create KUBE_CONFIG_AZURE in Repository secrets.
3. For value, Base64 string of the kubeconfig is expected, get it with: `cat $HOME/.kube/config | base64`
4. Paste the Base64 string and save the secret. There should be no empty line at the end.
5. Add `AAD_SERVICE_PRINCIPAL_CLIENT_ID` and `AAD_SERVICE_PRINCIPAL_CLIENT_SECRET` as well.
6. Push to `develop` branch to trigger staging pipeline.

Do all of the steps for frontend repo as well.
