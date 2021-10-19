# ArgoCD 

In their own words:

> Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

To learn more about ArgoCD visit their [website](https://argo-cd.readthedocs.io/en/stable/)

## ArgoCD Helm Chart

In order to deploy ArgoCD, we will use its helm charts. They are available at the following github 
[repo](https://github.com/argoproj/argo-helm)

## App of Apps

The deployment of ArgoCD and Dandelion, follows the [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) 
approach.

This means that the ArgoCD will first install itself and then proceed to install all the required components.

## Bootstrap

Before bootstrapping ArgoCD, it's necessary to fetch all Helm dependencies.

> **NOTE**: Please check section below on how to set a custom password to access ArgoCD. 

```shell
APP_PROVIDER=k3s
APP_NETWORK=testnet

export ARGOCD_PASSWORD=CH4NG3@M3
export ARGOCD_HASHED_PASSWORD=$(htpasswd -nbBC 10 null $ARGOCD_PASSWORD | sed 's|null:\(.*\)|\1|g')


cd argocd-bootstrap

helm dependency update
helm dependency build

helm upgrade \
    --create-namespace \
    --namespace argocd \
    --install argocd \
    --set "argo-cd.configs.secret.argocdServerAdminPassword=${ARGOCD_HASHED_PASSWORD}" \
    --set "argo-cd.configs.secret.argocdServerAdminPasswordMtime=$(date +%FT%T%Z)" \
    -f values-scaleway.yaml \
    -f values-scaleway-mainnet-stakeboard.yaml \
    .
```

## Deploy more apps/networks

* Install `argocd` cli tool and login into the server
* Set environment:
```
APP_PROVIDER=k3s
APP_NETWORK=mainnet
APP_SUFFIX=-v9-0-0
APP_NAME=dandelion-${APP_NETWORK}${APP_SUFFIX}
APP_REVISION=refactor/add-base-and-multi-node-overlays # this can be a branch
VALUES_FILE=argocd-bootstrap/values-${APP_PROVIDER}-${APP_NETWORK}.yaml
```
* Then execute from this from the root of the repository:
```
argocd app create \
  ${APP_NAME} \
  --project default \
  --repo https://gitlab.com/gimbalabs/dandelion/kustomize-dandelion.git \
  --revision ${APP_REVISION} \
  --path applications/main-app \
  --helm-set git.targetRevision=${APP_REVISION} \
  --values-literal-file ${VALUES_FILE} \
  --sync-policy automated \
  --sync-option Prune=true \
  --sync-option selfHeal=true \
  --sync-option ApplyOutOfSyncOnly=true \
  --dest-server https://kubernetes.default.svc
```

## Access ArgoCD

ArgoCD requires username and password authentication. By default, at the moment of deploying ArgoCD for the first time, username is `admin` and password is set to the full pod name of the `argocd-server`.
You can easily find the password and access the service by following these steps:

* Get default username/password:
```bash
POD_NAME=$(kubectl get pod -n argocd --selector 'app.kubernetes.io/name=argocd-server' --template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo -e "Username: admin\nPassword: ${POD_NAME}"
```
* Expose ArgoCD server port locally on your machine so you can access it via http://localhost:8080:
```bash
kubectl port-forward -n argocd ${POD_NAME} 8080:8080
```

Alternatively you can set the password at bootstrap time. Steps are:

1. Generate a new password
2. Use the bcrypt to hash the password. You can use this convenience [website](https://www.browserling.com/tools/bcrypt) to do so or `htpasswd` from the `apache2-utils` suite like this:
```
ARGOCD_PASSWORD=CH4NG3@M3
ARGOCD_HASHED_PASSWORD=$(htpasswd -nbBC 10 null $ARGOCD_PASSWORD | sed 's|null:\(.*\)|\1|g')
```
3. Specify the password when bootstrapping the cluster by replacing `argo-cd.configs.secret.argocdServerAdminPassword` hash:
```bash
helm upgrade \
    --create-namespace \
    --namespace argocd \
    --install argocd \
    --set "argo-cd.configs.secret.argocdServerAdminPassword=${ARGOCD_HASHED_PASSWORD}" \
    --set "argo-cd.configs.secret.argocdServerAdminPasswordMtime=$(date +%FT%T%Z)" \
    -f values-${APP_PROVIDER}-${APP_NETWORK}.yaml \
    .
```
## ArgoCD FAQ

The ArgoCD website has a great [FAQ Section](https://argo-cd.readthedocs.io/en/stable/faq/)
