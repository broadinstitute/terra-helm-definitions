# AP ArgoCD

This directory contains documentation and a values file for configuring the [AP Devops ArgoCD instance](https://ap-argocd.dsp-devops.broadinstitute.org/applications).

## Pushing Values Changes

Here's how to push updates to values.yaml to the existing, deployed ArgoCD installation:

    helm upgrade ap-argocd argo-cd \
      --repo https://argoproj.github.io/argo-helm \
      --version 2.3.5 \
      --namespace ap-argocd \
      -f ./values.yaml

## Initial deployment / Manual setup

Configure kubectl

    gcloud container clusters get-credentials dsp-tools --project=dsp-tools-k8s

Create namespace

    kubectl create namespace ap-argocd

Create TLS secret using pre-made tls crt & key files

    kubectl -n ap-argocd create secret generic ap-argocd-ingress-tls \
      --from-file=tls.crt \
      --from-file=tls.key

Create Vault secret for repo server (pull values from secret/suitable/ap-argocd/approle)

    kubectl -n ap-argocd create secret generic ap-argocd-reposerver-vault \
      --from-literal=roleid=<role id>
      --from-literal=secretid=<secret id>

Deploy preinstall chart (no values file needed)

    helm install ap-argocd-preinstall ap-argocd-preinstall \
      --repo https://broadinstitute.github.io/terra-helm \
      --version 0.0.1 \
      --namespace ap-argocd

Deploy official ArgoCD chart

    helm install ap-argocd argo-cd \
      --repo https://argoproj.github.io/argo-helm \
      --version 2.2.2 \
      --namespace ap-argocd \
      -f ./values.yaml

Configure [GitHub OAuth](https://argoproj.github.io/argo-cd/operator-manual/user-management/#dex)

    # Base64-encode the GitHub OAuth client secret
    echo -n <client secret> | base64

    # Take output and add to argocd-secret under the key dex.github.clientSecret
    kubectl edit secret -n ap-argocd argocd-secret

### Change Local Admin password

    # Get current admin password (it's the pod name)
    kubectl get pods -n ap-argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2

    # Set up port forwarding
    kubectl port-forward service/ap-argocd-server -n ap-argocd 8080:443

    # Log in and change password
    argocd login localhost:8080 --username admin
    argocd account update-password

    # Update admin password in Vault
    vault write secret/suitable/ap-argocd/local-accounts/admin \
      username=admin password=<configured-password>

### Generate and save API token for terra-ci account

    # Generate API token
    argocd account generate-token --account terra-ci

### Configure Projects & Applications

These steps will be migrated to a declarative configuration at some point, but for now:

    # Add terra-dev GKE cluster
    argocd cluster add gke_broad-dsde-dev_us-central1-a_terra-dev

    # Configure repository
    argocd repo add https://github.com/broadinstitute/terra-env-helm \
       --name terra-env-helm --type git \
       --username ignored --password <GITHUB TOKEN>

    # Create new project for non-prod Terra environments
    argocd proj create terra --description "Non-production test/integration environments for Terra"

    # Add helm chart repo as a source
    argocd proj add-source terra https://github.com/broadinstitute/terra-env-helm

    # Add terra-dev cluster, with terra-dev namespace, as a destination
    argocd proj add-destination terra https://35.238.186.116 terra-dev

    # Configure terra-env-helm application
    argocd app create terra-dev --project terra \
    --repo https://github.com/broadinstitute/terra-env-helm \
    --values values.yaml --values environments/dev.yaml --path . \
    --dest-namespace terra-dev --dest-server https://35.238.186.116

    # Configure firecloud-develop sync application
    argocd repo add https://github.com/broadinstitute/firecloud-develop \
       --name terra-env-helm --type git \
       --username ignored --password <GITHUB TOKEN>

    argocd proj add-source terra https://github.com/broadinstitute/firecloud-develop

    argocd app create terra-dev-configs --project terra \
      --repo https://github.com/broadinstitute/firecloud-develop \
      --revision dev \
      --dest-namespace terra-dev \
      --dest-server https://35.238.186.116 \
      --config-management-plugin legacy-consul-template \
      --path .
