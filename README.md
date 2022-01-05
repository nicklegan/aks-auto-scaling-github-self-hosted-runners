# AKS auto-scaling for GitHub self-hosted runners on Azure

> This documentation provides a step-by-step guide how to configure AKS auto-scaling for GitHub self-hosted runners. This is established by configuring the [actions-runner-controller](https://github.com/actions-runner-controller/actions-runner-controller).

The below auto-scaling guide consists of the following self-hosted runner specification:

- Optimized for Azure Kubernetes Service
- Compatible with GitHub Server and Cloud
- Organization-level runners
- Ephemeral runners
- Auto-scaling with **workflow_job** webhooks
- Webhook secret
- Ingress TLS termination
- Auto-provisioning Let's Encrypt SSL certificate
- GitHub App API authentication

## Table of Content

- [Prerequisites](#Prerequisites)
- [Setup AKS Cluster](#Setup-AKS-Cluster)
- [Setup Helm client](#Setup-Helm-client)
- [Add cert-manager and NGINX ingress repositories](#Add-cert-manager-and-NGINX-Ingress-repositories)
- [Install cert-manager](#Install-cert-manager)
- [Apply Let's Encrypt ClusterIssuer config for cert-manager](#Apply-Lets-Encrypt-ClusterIssuer-config-for-cert-manager)
- [Install NGINX ingress controller](Install-NGINX-ingress-controller)
- [Setup domain A record](#Setup-domain-A-record)
- [Create a GitHub App and configure GitHub App authenthication](#Create-a-GitHub-App-and-configure-GitHub-App-authenthication)
  - [Configure workflow_job webhooks](#Configure-workflow_job-webhooks)
  - [Generate and set GitHub App webhook secret](#Generate-and-set-GitHub-App-webhook-secret)
- [Prepare Actions Runner Controller configuration](#Prepare-Actions-Runner-Controller-configuration)
  - [values.yaml](#valuesyaml)
- [Install Actions Runner Controller](#Install-Actions-Runner-Controller)
- [Deploy runner manifest](#Deploy-runner-manifest)
  - [runnerdeployment.yaml](#runnerdeploymentyaml)
- [Verify deployment of all cluster services](#Verify-deployment-of-all-cluster-services)
- [Verify status of runners and pods](#Verify-status-of-runners-and-pods)
- [Resources](#Resources)

## Prerequisites

- [An Azure subscription](https://azure.microsoft.com/account)
- [GitHub Enterprise Server 3.3](https://github.blog/2021-12-07-github-enterprise-server-3-3-is-generally-available) or GitHub Enterprise Cloud
- A top-level domain name _(In this guide the example subdomain webhook.tld.com will be used)_

## Reference architecture

<img width="1583" alt="reference-architecture" src="https://user-images.githubusercontent.com/60080580/148239847-db32352e-50f2-40f6-9f9e-a82238c30374.png">

## Setup AKS cluster

```sh
# Install Azure CLI - https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
az login

# Install kubectl - https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az_aks_install_cli
az aks install-cli

# Create resource group
az group create -n <your-resource-group> --location <your-location>

# Create AKS cluster
az aks create -n <your-cluster-name> -g <your-resource-group> --node-resource-group <your-node-resource-group-name> --enable-managed-identity 

# Get AKS access credentials
az aks get-credentials -n <your-cluster-name> -g <your-resource-group>
```

## Setup Helm client

```sh
# Install Helm - https://helm.sh/docs/intro/install/
brew install helm # macOS
choco install kubernetes-helm # Windows
sudo snap install helm --classic # Debian/Ubuntu
```

## Add cert-manager and NGINX ingress repositories

```sh
# Add repositories
helm repo add jetstack https://charts.jetstack.io
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update repositories
helm repo update
```

## Install cert-manager

```sh
# Install cert-manager - https://cert-manager.io/docs/installation/helm/
helm install --wait --create-namespace --namespace cert-manager cert-manager jetstack/cert-manager --version v1.6.1 --set installCRDs=true
```

## Apply Let's Encrypt ClusterIssuer config for cert-manager

```sh
kubectl apply -f clusterissuer.yaml
```

### clusterissuer.yaml

- `email:`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: your-email@address.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            class: nginx
```

## Install NGINX ingress controller

```sh
# Install NGINX Ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace actions-runner-system --create-namespace

# Retrieve public load balancer IP from ingress controller
kubectl -n actions-runner-system get svc
```

## Setup domain A record

Navigate to your domain registrar and create a new A record linking the above ingress load balancer IP to your TLD as a subdomain. **e.g. webhook.tld.com**

## Create a GitHub App and configure GitHub App authenthication

- [Deploy using GitHub App Authentication](https://github.com/actions-runner-controller/actions-runner-controller#deploying-using-github-app-authentication)

### Configure workflow_job webhooks

- Activate the GitHub App webhook feature and add your earlier created domain A record as a Webhook URL
- Navigate to **permissions & events** and enable webhook [workflow job](https://docs.github.com/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#workflow_job) events

### Generate and set a GitHub App webhook secret

Prepare a webhook secret for use in the values.yaml file `github_webhook_secret_token` and configure the same webhook secret in the created GitHub App

```sh
# Generate random webhook secret
ruby -rsecurerandom -e 'puts SecureRandom.hex(20)'
```

## Prepare Actions Runner Controller configuration

Modify the default [values.yaml](https://github.com/actions-runner-controller/actions-runner-controller/blob/master/charts/actions-runner-controller/values.yaml) with your custom values like specified below

```sh
# Configure values.yaml
vim values.yaml
```

### values.yaml

- `githubEnterpriseServerURL:` only needed when using GHES
- `authSecret:`
- `githubWebhookServer:`
  - `ingress:`
  - `github_webhook_secret_token`

```yaml
# The URL of your GitHub Enterprise server, if you're using one.
githubEnterpriseServerURL: https://github.example.com

# Only 1 authentication method can be deployed at a time
# Uncomment the configuration you are applying and fill in the details
authSecret:
  create: true
  name: "controller-manager"
  annotations: {}
  ### GitHub Apps Configuration
  ## NOTE: IDs MUST be strings, use quotes
  github_app_id: "3"
  github_app_installation_id: "1"
  github_app_private_key: |-
    -----BEGIN RSA PRIVATE KEY-----
    MIIEogIBAAKCAQEA2zl6z+uMcS4D+D9f1ENLJY2w/9lLPajs/wA2gnt74/7bcB1f
    0000000000000000000000000000000000000000000000000000000000000000
    0000000000000000000000000000000000000000000000000000000000000000
    0000000000000000000000000000000000000000000000000000000000000000
    0000000000000000000000000000000000000000000000000000000000000000
    2x/9kVAWKQ2UJGxqupGqV14vLaNpmA2uILBxc5jKXHu1nNkgUwU=
    -----END RSA PRIVATE KEY-----
  ### GitHub PAT Configuration
  #github_token: ""
```

```yaml
githubWebhookServer:
  enabled: true
  replicaCount: 1
  syncPeriod: 10m
  secret:
    create: false
    name: "github-webhook-server"
    ### GitHub Webhook Configuration
    github_webhook_secret_token: ""
  imagePullSecrets: []
  nameOverride: ""
  fullnameOverride: ""
  serviceAccount:
    # Specifies whether a service account should be created
    create: true
    # Annotations to add to the service account
    annotations: {}
    # The name of the service account to use.
    # If not set and create is true, a name is generated using the fullname template
    name: ""
  podAnnotations: {}
  podLabels: {}
  podSecurityContext: {}
  # fsGroup: 2000
  securityContext: {}
  resources: {}
  nodeSelector: {}
  tolerations: []
  affinity: {}
  priorityClassName: ""
  service:
    type: ClusterIP
    annotations: {}
    ports:
      - port: 80
        targetPort: http
        protocol: TCP
        name: http
        #nodePort: someFixedPortForUseWithTerraformCdkCfnEtc
  ingress:
        enabled: true
        annotations:
          kubernetes.io/ingress.class: nginx
          cert-manager.io/cluster-issuer: "letsencrypt-prod"
        hosts:
          - host: webhook.tld.com
            paths:
            - path: /
        tls:
          - secretName: letsencrypt-prod
            hosts:
              - webhook.tld.com
```

## Install Actions Runner Controller

```sh
# Install actions-runner-controller
helm upgrade --install -f values.yaml --wait --namespace actions-runner-system actions-runner-controller actions-runner-controller/actions-runner-controller
```

## Verify installation and SSL certificate

```sh
# View all namespace resources
kubectl --namespace actions-runner-system get all

# Verify certificaterequest status
kubectl get certificaterequest --namespace actions-runner-system

# Verify certificate status
kubectl describe certificate letsencrypt --namespace actions-runner-system

# Verify if SSL certificate is working properly
curl -v --connect-to webhook.tld.com https://webhook.tld.com
```

## Deploy runner manifest

```sh
# Create a new namespace
kubectl create namespace self-hosted-runners

# Edit runnerdeployment yaml
vim runnerdeployment.yaml

# Apply runnerdeployment manifest
kubectl apply -f runnerdeployment.yaml
```

### runnerdeployment.yaml

The below manifest deploys organization-level auto-scaling ephemeral runners, using a minimal keep-alive configuration of 1 runner. Runners are scaled up to 5 active replicas based on incoming **workflow_job** webhook events. Scaling them back down to 1 runner by idle timeout of 5 minutes

- `organization:`

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: org-runner
  namespace: self-hosted-runners
spec:
  template:
    metadata:
      labels:
        app: org-runner
    spec:
      organization: your-github-organization
      labels:
        - self-hosted
      ephemeral: true
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: org-runner
  namespace: self-hosted-runners
spec:
  scaleTargetRef:
    name: org-runner
  scaleUpTriggers:
    - githubEvent: {}
      amount: 1
      duration: "5m"
  minReplicas: 1
  maxReplicas: 5
```

## Verify status of runners and pods

```sh
# List running pods
kubectl get pods -n self-hosted-runners

# List active runners
kubectl get runners -n self-hosted-runners
```

## Verify deployment of all cluster services

```sh
kubectl get all -A 
```
## Resources

- [actions-runner-controller](https://github.com/actions-runner-controller/actions-runner-controller)
- [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/azure/aks/)
- [Create Kubernetes secret for the TLS certificate](https://docs.microsoft.com/en-us/azure/aks/ingress-own-tls#create-kubernetes-secret-for-the-tls-certificate) (configure a custom SSL certificate)
- [Securing NGINX-ingress](https://cert-manager.io/docs/tutorials/acme/ingress/)
- [Troubleshooting Issuing ACME Certificates](https://cert-manager.io/docs/faq/acme/)
