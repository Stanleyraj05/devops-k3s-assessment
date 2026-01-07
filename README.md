# Infrastructure DevOps Assessment - Synthlane  
**Author:** Stanley Raj Kumar  
**Role:** Infrastructure DevOps Intern

---

## Overview
This repository contains the solution for the Infrastructure Owner assessment.  
The goal was to:

- Set up a production-ready environment on a Hetzner Cloud VM
- Deploy Open WebUI using Helm
- Configure OIDC authentication
- Debug a deliberate SSL failure
- Demonstrate judgment and ownership

---

## Part 1 & 2: Cluster Setup and Deployment

### 🖥️ 1. Environment Setup (Hetzner VM)

**OS:** Ubuntu 24.04  
**Cluster:** k3s (Single Node)  
**Container Runtime:** Docker

```bash
# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker

# Install k3s
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
 2. Application Deployment (Open WebUI)
Deployed Open WebUI using the official Helm chart.

bash
Copy code
# Add Repo
helm repo add open-webui https://helm.openwebui.com/
helm repo update

# Install Chart (Initial)
helm upgrade --install webui open-webui/open-webui \
  --namespace openwebui \
  --create-namespace \
  --set service.type=ClusterIP
Part 3 & 4: Debugging the SSL Failure (Intentional)
 Issue Summary
After enabling OIDC authentication pointing to:

arduino
Copy code
https://46.62.233.25/auth/realms/hyperplane/
Open WebUI failed authentication due to SSL validation failure.

 Diagnosis
Checked logs
No crash — only connection warnings.

Reproduced inside pod
Manually tested Python requests from inside pod:

kubectl exec -it -n openwebui open-webui-0 -- /bin/bash
python -c "import requests; print(requests.get('https://46.62.233.25/auth/realms/hyperplane/.well-known/openid-configuration').text)"

Error Returned

yaml
ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED]
certificate verify failed: self-signed certificate
 Root Cause
OIDC provider (Keycloak) uses a self-signed certificate.
Python requests rejects untrusted certs by default.

 Fix — Production-Grade Solution
Instead of disabling SSL verification, I added the custom CA to the container trust store.

Step 1 — Fetch Certificate

openssl s_client -showcerts -connect 46.62.233.25:443 </dev/null 2>/dev/null \
  | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > oidc-ca.crt

Step 2 — Create Kubernetes Secret

kubectl create secret generic oidc-ca \
  --from-file=oidc-ca.crt=oidc-ca.crt \
  -n openwebui
Step 3 — Helm Override (values-oidc.yaml)
yaml
Copy code
extraVolumes:
  - name: oidc-ca-volume
    secret:
      secretName: oidc-ca

extraVolumeMounts:
  - name: oidc-ca-volume
    mountPath: /etc/ssl/certs/oidc-ca.crt
    subPath: oidc-ca.crt
    readOnly: true

extraEnv:
  - name: REQUESTS_CA_BUNDLE
    value: /etc/ssl/certs/oidc-ca.crt
  - name: SSL_CERT_FILE
    value: /etc/ssl/certs/oidc-ca.crt

Step 4 — Deploy Patch

helm upgrade webui open-webui/open-webui \
  -n openwebui \
  -f values-oidc.yaml
 OIDC successfully authenticated after rollout.

Part 5: Ownership & Architecture Answers
 1. Production Readiness
Top 5 Risks:
Single point of failure (one-node k3s)

Local storage — risk of data loss

Manual secret management (no Vault/ESO)

No monitoring/alerting system

No ingress for TLS/traffic routing

First 2 things to fix:
Implement backups (Velero/S3 snapshots)

Deploy Traefik or Nginx + Cert-Manager for Let's Encrypt certs

 2. Failure Scenario — Traffic Spike & Node Down at 2AM
What breaks first:
CPU/memory → OOMKill → API server unavailable.

Immediate recovery:

Reboot via Hetzner console

Scale VM vertically

Long-term fix:

Move to HA cluster with multiple worker nodes

Add Horizontal Pod Autoscaling (HPA)

Add Load Balancer in front

 3. Secret Management
Proper method:
External Secrets Operator (AWS SM, Vault)

Sealed Secrets (GitOps)

Never commit to Git:
API keys

Client secrets

Private keys

Database passwords

Must rotate regularly:
OIDC client secrets

SSH keys

DB credentials

 4. Backup & Recovery
What to back up:
PVCs (Vector DB, user data)

Helm values and cluster config

Frequency:
Daily full + WAL logs

Backup config every commit

Verification:
Monthly “Game Day”

Restore into fresh cluster and validate

 5. Cost Ownership (Hetzner)
Keep costs low:
Use Hetzner Cloud resources efficiently

Right-size VM

Avoid over-engineering

Use standard storage tiers

Move away from k3s when:
Compliance required

HA control plane needed

Team size increases

 Required Outputs
kubectl get nodes
pgsql
Copy code
NAME             STATUS   ROLES                  AGE     VERSION
rstaneleyraj05   Ready    control-plane,master   3h20m   v1.30.2+k3s1

kubectl get all -n openwebui
NAME                                       READY   STATUS    RESTARTS   AGE
pod/open-webui-0                           1/1     Running   0          5m
pod/open-webui-ollama-5d99896fd7-f59xv     1/1     Running   0          3h
pod/open-webui-pipelines-7d8757f9c-l74z9   1/1     Running   0          3h
pod/open-webui-redis-c47dbfbcd-sz7c7       1/1     Running   0          3h

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)       AGE
service/open-webui             ClusterIP   10.43.150.23    <none>        80/TCP        3h
service/open-webui-ollama      ClusterIP   10.43.200.10    <none>        11434/TCP     3h
service/open-webui-pipelines   ClusterIP   10.43.10.55     <none>        9099/TCP      3h
service/open-webui-redis       ClusterIP   10.43.88.99     <none>        6379/TCP      3h

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/open-webui-ollama      1/1     1            1           3h
deployment.apps/open-webui-pipelines   1/1     1            1           3h
deployment.apps/open-webui-redis       1/1     1            1           3h

NAME                          READY   AGE
statefulset.apps/open-webui   1/1     3h
