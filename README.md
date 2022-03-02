
# GitOps-Service Infrastructure Deployments

This repository is an initial set of Argo-CD-based deployments of GitOps components to a cluster, plus a script to bootstrap Argo CD onto that cluster (to drive these Argo-CD-based deployments, via OpenShift GitOps).
This repository is structured as a GitOps monorepo (e.g. the repository contains the K8s resources for *multiple* applications), using [Kustomize](https://kustomize.io/).

## Maintaining gitops components

Simply update the files under `components/gitops`, and open a PR with the changes. 

## Bootstrapping a cluster
### Required prerequisites
The prerequisites are:
- You must have `kubectl`, `oc`, `jq`, `yq` and `kustomize` installed. 
- You must have `kubectl` and `oc` pointing to an existing OpenShift cluster, that you wish to deploy to.

### Bootstrap GitOps-Service
Steps:
1) Run `./hack/bootstrap-cluster.sh` which will bootstrap Argo CD (using OpenShift GitOps) and setup the Argo CD `Application` Custom Resources (CRs) for gitops component. This command will output the Argo CD Web UI route when it's finished.
2) Open the Argo CD Web UI to see the status of your deployments. You can use the route from the previous step and login using your OpenShift credentials (using the 'Login with OpenShift' button), or login to the OpenShift Console and navigate to Argo CD using the OpenShift Gitops menu in the Applications pulldown.
![OpenShift Gitops menu with Cluster Argo CD menu option](documentation/images/argo-cd-login.png?raw=true "OpenShift Gitops menu")
3) If your deployment was successful, you should see several applications running, such as "all-components-staging", "gitops".

#### Post-bootstrap GitOps Service Configuration

GitOps service components will not be functional right after bootstrap. It requires manual configuration in order to work properly:

1) `wget https://raw.githubusercontent.com/redhat-appstudio/managed-gitops/main/manifests/postgresql-staging/postgresql-staging-secret.yaml`

2) Edit `postgresql-staging-secret.yaml`, and update the password field:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitops-postgresql-staging
  labels:
    app.kubernetes.io/name: postgresql
    helm.sh/chart: postgresql-10.16.1
    app.kubernetes.io/instance: gitops-postgresql-staging
    app.kubernetes.io/managed-by: Helm
  namespace: gitops
type: Opaque
data:
  postgresql-password: "(your password here)" # Edit this line
```

*Note*: You will need to use a [Base 64 value](https://www.base64encode.org/) for the password field. (Note: this password is *not* required after this step, so you may safely discard it after editing the secret.)

3) `kubectl apply -f postgresql-staging-secret.yaml`  to apply the Secret YAML.

4) You may need to hit 'Synchronize' on the `gitops` application, in Argo CD, in order to trigger the updated Application deployment.
    - (I'm not 100% sure if this is required, more data are needed, but it can't hurt! - @jgwest)

# Invoking the API

## GitOps Service

Once the cluster is successfully bootstrapped, create a Namespace with the `argocd.argoproj.io/managed-by: gitops-service-argocd` label, for example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: (your-user-name)
  labels:
    argocd.argoproj.io/managed-by: gitops-service-argocd
```

The `argocd.argoproj.io/managed-by: gitops-service-argocd` label gives 'permission' to Argo CD (specifically, the instance in `gitops-service-argocd`) to deploy to your namespace.

You may now create `GitOpsDeployment` resources, which the GitOps Service will respond to, deploying resources to your namespace:
```yaml
apiVersion: managed-gitops.redhat.com/v1alpha1
kind: GitOpsDeployment

metadata:
  name: gitops-depl
  namespace: (your-user-name)

spec:

  # Application/component to deploy
  source:
    repoURL: https://github.com/redhat-appstudio/gitops-repository-template
    path: environments/overlays/dev

  # destination: {}  # destination is user namespace if empty

  # Only 'automated' type is currently supported: changes to the GitOps repo immediately take effect (as soon as Argo CD detects them).
  type: automated
```


See the [GitOps Service M2 Demo script for more details](https://github.com/redhat-appstudio/managed-gitops/tree/main/examples/m2-demo#run-the-demo).

