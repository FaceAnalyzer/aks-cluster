<p align="center">
    <img src="https://github.com/cncf/artwork/blob/main/projects/kubernetes/icon/color/kubernetes-icon-color.png" width="20%">
</p>

# AKS cluster for Faceanalyzer

This repo contains configuration for setting up an AKS cluster to which Faceanalyzer application can be deployed. AKS is a Kubernetes solution offered by Azure. FaceAnalyzer can be deployed to any Kubernetes cluster, even the bare metal.

First you need to create an AKS cluster with the Azure dashboard. Then, you need to install Ingress-nginx and Cert-manager so that our app can be accesible from the public Internet. This repo is organized in IaC way and you can install the services using [Helmfile](https://github.com/helmfile/helmfile). You will also need [Kubectl](https://kubernetes.io/docs/tasks/tools/) and [Helm](https://helm.sh/docs/intro/install/).

## 1. Create AKS cluster on Azure

1. Open the [Azure Portal](https://azure.microsoft.com/en-us/get-started/azure-portal), sign in, and to go [Kubernetes services](https://portal.azure.com/#view/HubsExtension/BrowseResource/resourceType/Microsoft.ContainerService%2FmanagedClusters). Here you can manage Azure Kubernetes Service (AKS) clusters.
2. Create a new Kubernetes cluster.
3. Choose a subscription for the cluster and a resource group. For Cluster preset configuration you can leave Dev/Test. Name the cluster, for example `faceanalyzer`, and choose a region. For Kubernetes version put `1.27.7`. For Authentication and Authorization put `Azure AD authentication with Azure RBAC`. Click next.
4. Now we have to define node pool configuration. A node pool is basically set of VMs that run Kubernetes nodes. Click on `agentpool` and set its Node size to `B2ms` (2 CPU, 8 GB RAM). For Scale method put manual and for Node count put 1. Save the node pool.
5. We can now jump to Review + Create and create the cluster.
6. After the cluster is created, open it in the dashboard, and add yourself to IAM. Create a new role assignment and for Role choose `Azure Kubernetes Service RBAC Cluster Admin`.

## 2. Get access to the cluster

1. Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) to your local terminal and run: `az login`.
1. Get the kubeconfig: `az aks get-credentials --resource-group faceanalyzer --name faceanalyzer`.
2. Install [Kubelogin](https://github.com/Azure/kubelogin): `az aks install-cli`.
2. Convert the kubeconfig with Kubelogin: `kubelogin convert-kubeconfig`.
2. Try running `kubectl get namespaces` to test cluster access.

You can now clone this Git repo to install Ingress-nginx and Cert-manager.

## 3. Install Ingress-nginx

Ingress-nginx is a reverse proxy. It enables external access to ingresses in cluster.

1. Open terminal and navigate to `ingress-nginx/` of this repo.
2. Run `helmfile apply`.
3. Run `kubectl get svc -n ingress-nginx -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'` to get IP adress of ingress-nginx.
4. Add the IP adress to the DNS provider as A record.

## 4. Install Cert-manager

Cert-manager issues Let's Encrypt certificates for TLS. This enables HTTPS for our app.

1. Open terminal and navigate to `cert-manager/` of this repo.
2. Run `helmfile apply`.

## 5. Create the namespace

Namespace for FaceAnalyzer application should be created manually.

```bash
kubectl create namespace faceanalyzer
```

## 6. Create service principal

Service principal is a non-interactive user. Our CI/CD on backend and frontend projects needs access to the cluster so that it can deploy our app. That's why we need a service principal for CI/CD.

1. Create `faceanalyzer-cicd` service principal if it doesn't exist. In any case, go to Azure Active Directory (Microsoft Entra ID) and then App registrations. `faceanalyzer-cicd` should be here. If it doesn't exist, just create it.
2. Go back to the Kubernetes cluster and add `faceanalyzer-cicd` service principal to IAM. Choose `Azure Kubernetes Service RBAC Cluster Admin` for the role.

## 7. Get kubeconfig as service principal

CI/CD uses kubeconfig to deploy Faceanalyzer application the cluster. This requires service principal on Azure. As already mentioned, CI/CD uses `faceanalyzer-cicd` principal.

Open the principal's page in App registrations to find its token. Tokens are located on Certificates & secrets page, under Client secrets. You should create a new token and save its value for the next step.

To get the principal's kubeconfig and test if it works, first get the kubeconfig and then convert it with prinicipal's credentials. ID refers to Application (client) ID, secret refers to secret token you've just created.

```bash
az aks get-credentials --resource-group faceanalyzer --name faceanalyzer
export AAD_SERVICE_PRINCIPAL_CLIENT_ID=824f57a1-191c-4ace-ac4e-09ff215a7cfe
export AAD_SERVICE_PRINCIPAL_CLIENT_SECRET=seCreTStrInG
kubelogin convert-kubeconfig -l spn
kubectl get namespaces
```

You should get a list of namespaces. You are now accesing the cluster as `faceanalyzer-cicd`.

Then, you need to add the kubeconfig to our CI/CD service, GitHub Actions.

1. Open backend repo and go to Settings > Secrets and Variables > Actions.
2. Create KUBE_CONFIG_AZURE in Repository secrets.
3. For value, Base64 string of the kubeconfig is expected, get it with: `cat $HOME/.kube/config | base64`
4. Paste the Base64 string and save the secret. There should be no empty line at the end.
5. Add `AAD_SERVICE_PRINCIPAL_CLIENT_ID` and `AAD_SERVICE_PRINCIPAL_CLIENT_SECRET` as well.
6. Push to `develop` branch to trigger staging pipeline.

Do all of the steps for frontend repo as well.

Finally, delete the local kubeconfig: `rm $HOME/.kube/config`.