---
author: ["Thariq Shah"]
title: "Building a Homelab with K3s"
date: 2025-05-13T22:48:10+05:30
draft: false
ShowToc: true
cover:
  image: /setting-up-k3s/cover/cover.png
categories: ["homelab", "k3s", "how-to"]
tags: ["homelab", K3s]
series: ["Home labs"]
searchHidden: true
---

# 🚀 Self-Hosting with K3s and Traefik

Set up a lightweight Kubernetes cluster using K3s on a Raspberry Pi, deploy Traefik via Helm, and prepare for self-hosted services with ingress routing and local TLS.

---

## 🧰 Prerequisites

- A Linux host with root access (e.g., Raspberry Pi or any ARM64/x86_64 board)
- `kubectl` installed on a client machine
- Helm 3 installed on the cluster (https://helm.sh/docs/intro/install/)

---

## ⚙️ Install K3s without Traefik

Skip the bundled Traefik and gain full control by managing ingress via Helm

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

### Verify k3s installation

Confirm the cluster is operational

```bash
kubectl get nodes
```

---

## 🖧 Configuring kubectl for remote access in same network

Install kubectl from https://kubernetes.io/docs/tasks/tools/

Next, copy the k3s.yaml file from your cluster’s machine to the .kube directory and rename it to config

```bash
mkdir -p ~/.kube
```

copy file `/etc/rancher/k3s/k3s.yaml` from cluster machine to client machine `~/.kube/config`

Finally, verify the connection from your client machine

```bash
kubectl get nodes
```

---

## 🤖 Installing Traefik with Helm

Add helm repo

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```

To organize the Traefik deployment, create a dedicated namespace

```bash
kubectl create namespace traefik
```

Create `values.yaml` for Traefik Configuration

```yaml
globalArguments:
  - "--global.sendanonymoususage=false"
  - "--global.checknewversion=false"
additionalArguments:
  - "--serversTransport.insecureSkipVerify=true"
  - "--log.level=INFO"
deployment:
  enabled: true
  replicas: 3
  annotations: {}
  podAnnotations: {}
  additionalContainers: []
  initContainers: []
ports:
  web:
    port: 80
    targetPort: 80
  websecure:
    port: 443
    targetPort: 443
ingressRoute:
  dashboard:
    enabled: false
providers:
  kubernetesCRD:
    enabled: true
    ingressClass: traefik-external
    allowExternalNameServices: true
  kubernetesIngress:
    enabled: true
    allowExternalNameServices: true
    publishedService:
      enabled: false
rbac:
  enabled: true
service:
  enabled: true
  type: LoadBalancer
  annotations: {}
  labels: {}
  spec:
    loadBalancerIP: 192.168.0.171 #local network ip
  loadBalancerSourceRanges: []
  externalIPs: []
```

Finally, use Helm to deploy Traefik with the custom configuration

```bash
helm install --namespace=traefik traefik traefik/traefik --values=values.yaml
```

---

## 🔧 Configuring Traefik

### Apply default headers as middleware for services

create a yaml file default-headers.yaml

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: default-headers
  namespace: default
spec:
  headers:
    browserXssFilter: true
    contentTypeNosniff: true
    forceSTSHeader: true
    stsIncludeSubdomains: true
    stsPreload: true
    stsSeconds: 15552000
    referrerPolicy: no-referrer
    contentSecurityPolicy: "default-src 'none'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:; img-src 'self' data: https:; font-src 'self' https: data:; connect-src 'self' https:; frame-src 'self' https:; media-src 'self' https:; object-src 'none'; frame-ancestors 'self'; base-uri 'self'; form-action 'self';"
    customFrameOptionsValue: SAMEORIGIN
    customRequestHeaders:
      X-Forwarded-Proto: https
```

```bash
kubectl apply -f default-headers.yaml
```

---

## 🔐 Protect the Traefik Dashboard

Generate a basic auth string

```bash
htpasswd -nb user strongpassword | base64
```

create a file secret-dashboard.yaml

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
type: Opaque
data:
  users: <base64-encoded-auth>
```

Apply

```bash
kubectl apply -f secret-dashboard.yaml
```

create a middleware for traefik dashboard

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: traefik-dashboard-basicauth
  namespace: traefik
spec:
  basicAuth:
    secret: traefik-dashboard-auth
```

```bash
kubectl apply -f middleware.yaml
```

### IngressRoute for Traefik Dashboards

create ingress.yaml

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik-dashboard.example.com`)
      kind: Rule
      middlewares:
        - name: traefik-dashboard-basicauth
          namespace: traefik
      services:
        - name: api@internal
          kind: TraefikService
```

```bash
kubectl apply -f ingress.yaml
```

---

## 👨‍💻 Access via Local DNS Mapping

To access the dashboard from your local network, update the /etc/hosts file on your machine

```bash
sudo nano /etc/hosts
```

Add the following line (replace with your cluster's IP address):

```text
GNU nano 8.3                       etc/hosts
# Loopback entries; do not change.
# For historical reasons, localhost precedes localhost.localdomain:
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
# See hosts(5) for proper format and other examples:
# 192.168.1.10 foo.example.org foo
# 192.168.1.13 bar.example.org bar
192.168.0.171 traefik-dashboard.example.com # ip of your cluster
```

Now, you can access the Traefik dashboard at `traefik-dashboard.example.com` in your browser.

---
