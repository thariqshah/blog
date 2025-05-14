---
author: ["Thariq Shah"]
title: "Securing K3s With Lets Encrypt"
date: 2025-05-14T19:55:46+05:30
draft: false
ShowToc: true
cover:
  image: /cert-manager-lets-encrypt/cover/cover.png
categories: ["homelab", "k3s", "how-to"]
tags: ["homelab", K3s]
series: ["k3s homelab"]
searchHidden: true
---

# 🔐 TLS Certificates for Your Homelab Using Cloudflare + Cert-Manager + Let's Encrypt

With Traefik operational as the ingress controller in a K3s cluster, the next logical step is to provision HTTPS automatically using **Let's Encrypt** certificates, **Cloudflare DNS validation**, and **Cert-Manager**. This ensures encrypted traffic across services in a self-hosted homelab.

## ☁️ Prerequisites

1. A domain managed by **Cloudflare**
2. An API token with DNS edit permissions
3. **Traefik** installed and configured as the ingress controller
4. A functional **K3s** cluster with outbound internet access

## 🔧 Step 1: Install Cert-Manager

Install `cert-manager` using Helm

```bash
kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.17/cert-manager.crds.yaml

kubectl get  certificates
```

Create a `values.yaml`

```yaml
installCRDs: false
replicaCount: 3
extraArgs:
  - --dns01-recursive-nameservers=1.1.1.1:53,9.9.9.9:53
  - --dns01-recursive-nameservers-only
podDnsPolicy: None
podDnsConfig:
  nameservers:
    - 1.1.1.1
    - 9.9.9.9
```

Install `cert-manager`

```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager --values=values.yaml --version v1.12.17
```

## 🔑 Step 2: Create a Cloudflare API Token

Generate an API token via Cloudflare Dashboard:

- Navigate to My Profile → API Tokens → Create Token

- Use the Edit zone DNS template

- Scope the token to either All Zones

- Save the token securely

## 🧾 Step 3: Store Cloudflare Token as Kubernetes Secret

Create a secret manifest (`secret-cf-token.yaml`)

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  cloudflare-token: <cloudlare-api-token>
```

```bash
kubectl apply -f secret-cf-token.yaml
```

## 📄 Step 4: Configure a Let's Encrypt ClusterIssuer (Staging)

Start with Let's Encrypt staging environment for testing:

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: example@account.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - dns01:
          cloudflare:
            email: example@account.com
            apiTokenSecretRef:
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "example.com"
```

```bash
kubectl apply -f letsencrypt-issuer-staging.yaml
```

## 🔐 Step 5: Request a Certificate

Issue a wildcard TLS certificate for the domain

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-staging-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: "*.example.com"
  dnsNames:
    - "example.com"
    - "*.example.com"
```

```bash
kubectl apply -f example-com-cert.yaml
```

## 🚪 Step 6: Enable TLS for Services

Configure Traefik’s IngressRoute to use the issued certificate

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik-dashboard.example.com`)
      kind: Rule
      middlewares:
        - name: traefik-dashboard-basicauth
      services:
        - name: api@internal
          kind: TraefikService
  tls:
    secretName: example-com-staging-tls
```

```bash
kubectl apply -f ingress.yaml
```

## 🚀 Switch to Let's Encrypt Production

Once certificate issuance through staging is successful, switch to the production environment.

Create the production ClusterIssuer

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: example@account.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - dns01:
          cloudflare:
            email: example@account.com
            apiTokenSecretRef:
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "example.com"
```

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-production-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "*.example.com"
  dnsNames:
    - "example.com"
    - "*.example.com"
```

```bash
kubectl apply -f example-com-cert.yaml

kubectl get certificates #should take a while for certs to be issued
```

##🌐 Final Step: Update DNS Records in Cloudflare
Create a CNAME or A record in Cloudflare pointing traefik-dashboard.example.com to your public IP or create cloudflared tunnels
