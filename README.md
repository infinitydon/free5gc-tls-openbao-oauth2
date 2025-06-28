# OpenBao + cert-manager for free5GC OAuth2 Certificates

## Overview

This guide integrates OpenBao with cert-manager to provision and manage certificates for free5GC's OAuth2 implementation in Kubernetes. We use cert-manager Certificate CRDs with OpenBao as the PKI backend, then mount the generated secrets directly into pods - avoiding CSI to prevent unnecessary certificate regeneration on pod restarts.

## Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│    OpenBao      │───▶│   cert-manager   │───▶│   free5GC NRF   │
│   PKI Engine    │    │   Certificate    │    │      Pod        │
│                 │    │      CRD         │    │ (secret mounted │
└─────────────────┘    └──────────────────┘    │   as volume)    │
                              │                 └─────────────────┘
                              ▼
                       ┌──────────────────┐
                       │  Kubernetes      │
                       │    Secret        │
                       └──────────────────┘
```

## Prerequisites

### Install Kubernetes Cluster

```bash
sudo snap install microk8s --classic --channel=1.32

sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
chmod 0700 ~/.kube
microk8s config > ~/.kubeconfig

microk8s enable hostpath-storage
microk8s enable community
microk8s enable multus

curl -LO https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
### 1. Install OpenBao
```bash
# Install OpenBao using Helm
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update

helm install openbao openbao/openbao \
  --namespace openbao-system \
  --create-namespace \
  --set server.dev.enabled=true \
  --set server.dev.devRootToken="root" \
  --set injector.enabled=false --version 0.16.1

# Download latest version 1.20.0
export VAULT_VERSION="1.20.0"
curl -LO "https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip"

sudo apt install unzip jq -y

# Extract and install
unzip vault_${VAULT_VERSION}_linux_amd64.zip
sudo mv vault /usr/local/bin/
sudo chmod +x /usr/local/bin/vault
```

### 2. Install cert-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.0/cert-manager.yaml
```

### 3.  OpenBao PKI Setup
Initialize OpenBao PKI

# Option 1: From Local Machine (with jq available)
bash# Port forward to OpenBao
kubectl port-forward -n openbao-system svc/openbao 8200:8200 &

# Set environment
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="your-secure-token"  # NOT "root" in production

# Enable PKI engines
vault secrets enable -path=pki pki
vault secrets enable -path=pki_int pki

# Configure PKI engines
vault secrets tune -max-lease-ttl=87600h pki
vault secrets tune -max-lease-ttl=43800h pki_int

# Generate root CA
vault write -field=certificate pki/root/generate/internal \
    common_name="free5GC Root CA" \
    issuer_name="free5gc-root-ca" \
    ttl=87600h

# Create intermediate CA
vault write -format=json pki_int/intermediate/generate/internal \
    common_name="free5GC NRF Intermediate CA" \
    issuer_name="free5gc-nrf-intermediate" \
    | jq -r '.data.csr' > pki_intermediate.csr

vault write -format=json pki/root/sign-intermediate \
    issuer_ref="free5gc-root-ca" \
    csr=@pki_intermediate.csr \
    format=pem_bundle ttl="43800h" \
    | jq -r '.data.certificate' > intermediate.cert.pem

vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem

# Configure URLs
vault write pki_int/config/urls \
    issuing_certificates="$VAULT_ADDR/v1/pki_int/ca" \
    crl_distribution_points="$VAULT_ADDR/v1/pki_int/crl"

# Create role for all NFs (including NRF)
vault write pki_int/roles/free5gc-nf \
    issuer_ref="$(vault read -field=default pki_int/config/issuers)" \
    allowed_domains="free5gc.local,cluster.local" \
    allow_subdomains=true \
    allow_bare_domains=true \
    allow_localhost=true \
    allow_ip_sans=true \
    max_ttl="2160h" \
    generate_lease=true

# Option 2: Directly in OpenBao Pod (no jq needed)

Execute commands directly in OpenBao pod

kubectl exec -it -n openbao-system openbao-0 -- /bin/sh

# Inside the pod, set up environment
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="your-secure-token"

# Enable PKI engines
vault secrets enable -path=pki pki
vault secrets enable -path=pki_int pki

# Configure PKI engines
vault secrets tune -max-lease-ttl=87600h pki
vault secrets tune -max-lease-ttl=43800h pki_int

# Generate root CA
vault write -field=certificate pki/root/generate/internal \
    common_name="free5GC Root CA" \
    issuer_name="free5gc-root-ca" \
    ttl=87600h

# Create intermediate CA (without jq - save to file)
vault write -field=csr pki_int/intermediate/generate/internal \
    common_name="free5GC NRF Intermediate CA" \
    issuer_name="free5gc-nrf-intermediate" > /tmp/pki_intermediate.csr

# Sign intermediate with root CA (without jq - save to file)
vault write -field=certificate pki/root/sign-intermediate \
    issuer_ref="free5gc-root-ca" \
    csr=@/tmp/pki_intermediate.csr \
    format=pem_bundle ttl="43800h" > /tmp/intermediate.cert.pem

# Set the intermediate certificate
vault write pki_int/intermediate/set-signed certificate=@/tmp/intermediate.cert.pem

# Configure URLs
vault write pki_int/config/urls \
    issuing_certificates="http://127.0.0.1:8200/v1/pki_int/ca" \
    crl_distribution_points="http://127.0.0.1:8200/v1/pki_int/crl"

# Get the default issuer reference (without jq)
ISSUER_REF=$(vault read -field=default pki_int/config/issuers)

# Create role for all NFs (including NRF)
vault write pki_int/roles/free5gc-nf \
    issuer_ref="$ISSUER_REF" \
    allowed_domains="free5gc.local,cluster.local" \
    allow_subdomains=true \
    allow_bare_domains=true \
    allow_localhost=true \
    allow_ip_sans=true \
    max_ttl="2160h" \
    generate_lease=true

# Clean up temporary files
rm /tmp/pki_intermediate.csr /tmp/intermediate.cert.pem

# Exit the pod
exit

# OpenBao Kubernetes Authentication Setup (Modern Approach)
Enable Kubernetes auth

vault auth enable kubernetes

# Configure Kubernetes auth (modern method for K8s 1.21+)
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc.cluster.local" \
    disable_iss_validation=true

# Create policy for cert-manager
vault policy write cert-manager-policy - <<EOF
path "pki_int/issue/free5gc-nf" {
  capabilities = ["create", "update"]
}
path "pki/cert/ca" {
  capabilities = ["read"]
}
path "pki_int/cert/ca" {
  capabilities = ["read"]
}
EOF

# Create role for cert-manager service account (using existing cert-manager SA)
vault write auth/kubernetes/role/cert-manager \
    bound_service_account_names=cert-manager \
    bound_service_account_namespaces=cert-manager \
    policies=cert-manager-policy \
    audience=vault://openbao-issuer \
    ttl=24h

2. cert-manager Integration with OpenBao (Modern Kubernetes Auth)

kubectl create ns telco

kubectl apply -f openbao-issuers.yaml

3. Create Certificates
kubectl apply -f root-ca-certificate.yaml
kubectl apply -f all-nf-certificates.yaml

6. Certificate Monitoring and Troubleshooting

Certificate Status Monitor

yaml# cert-monitor.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cert-monitor
  namespace: free5gc
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-monitor
rules:
- apiGroups: ["cert-manager.io"]
  resources: ["certificates", "issuers", "clusterissuers"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets", "events"]
  verbs: ["get", "list", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-monitor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-monitor
subjects:
- kind: ServiceAccount
  name: cert-monitor
  namespace: free5gc
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cert-expiry-monitor
  namespace: free5gc
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cert-monitor
          containers:
          - name: cert-monitor
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              #!/bin/bash
              set -e
              
              echo "=== Certificate Status Report ==="
              echo "Timestamp: $(date)"
              echo
              
              # Check all certificates in free5gc namespace
              echo "Certificates in free5gc namespace:"
              kubectl get certificates -n free5gc -o custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type==\"Ready\")].status,RENEWAL:.status.renewalTime,EXPIRY:.status.notAfter
              echo
              
              # Check ClusterIssuers
              echo "ClusterIssuer Status:"
              kubectl get clusterissuers -o custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type==\"Ready\")].status
              echo
              
              # Check for certificates expiring within 7 days
              echo "Checking for certificates expiring within 7 days..."
              ALERT_FOUND=false
              
              for cert in $(kubectl get certificates -n free5gc -o jsonpath='{.items[*].metadata.name}'); do
                EXPIRY=$(kubectl get certificate $cert -n free5gc -o jsonpath='{.status.notAfter}' 2>/dev/null || echo "")
                if [ -n "$EXPIRY" ]; then
                  EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s 2>/dev/null || echo "0")
                  CURRENT_EPOCH=$(date +%s)
                  DAYS_LEFT=$(( (EXPIRY_EPOCH - CURRENT_EPOCH) / 86400 ))
                  
                  if [ $DAYS_LEFT -lt 7 ] && [ $DAYS_LEFT -gt 0 ]; then
                    echo "⚠️  ALERT: Certificate $cert expires in $DAYS_LEFT days ($EXPIRY)"
                    ALERT_FOUND=true
                  elif [ $DAYS_LEFT -le 0 ]; then
                    echo "🚨 CRITICAL: Certificate $cert has EXPIRED ($EXPIRY)"
                    ALERT_FOUND=true
                  fi
                fi
              done
              
              if [ "$ALERT_FOUND" = false ]; then
                echo "✅ All certificates are valid and not expiring soon"
              fi
              
              echo
              echo "=== OpenBao Health Check ==="
              if kubectl get pods -n openbao-system -l app.kubernetes.io/name=openbao | grep -q Running; then
                echo "✅ OpenBao is running"
              else
                echo "🚨 OpenBao is not running properly"
              fi
          restartPolicy: OnFailure

7. Deployment Scripts


# Deploy monitoring
echo "📊 Deploying certificate monitoring..."
kubectl apply -f cert-monitor.yaml

# Verification
echo "✅ Verifying deployment..."
echo
echo "ClusterIssuers:"
kubectl get clusterissuers
echo
echo "Certificates:"
kubectl get certificates -n ${NAMESPACE}
echo
echo "Secrets:"
kubectl get secrets -n ${NAMESPACE} | grep tls
echo
echo "Pods:"
kubectl get pods -n ${NAMESPACE}
echo

echo "🎉 Deployment complete!"
echo
echo "To check certificate details:"
echo "  kubectl describe certificate nrf-tls-cert -n ${NAMESPACE}"
echo
echo "To check NRF logs:"
echo "  kubectl logs -f deployment/nrf -n ${NAMESPACE}"
echo
echo "To run certificate monitoring manually:"
echo "  kubectl create job --from=cronjob/cert-expiry-monitor manual-cert-check -n ${NAMESPACE}"
8. Troubleshooting
Common Issues and Solutions
Issue: ClusterIssuer Not Ready
bash# Check ClusterIssuer status
kubectl describe clusterissuer openbao-issuer

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Verify OpenBao connectivity
kubectl run debug --rm -it --image=curlimages/curl -- \
  curl -k http://openbao.openbao-system.svc.cluster.local:8200/v1/sys/health
Issue: Certificate Not Issuing
bash# Check certificate status
kubectl describe certificate nrf-tls-cert -n free5gc

# Check CertificateRequest
kubectl get certificaterequests -n free5gc
kubectl describe certificaterequest <request-name> -n free5gc

# Check Vault authentication
vault write auth/kubernetes/login role=cert-manager jwt=<test-token>
Issue: Pod Authentication Failures
bash# Verify service account exists
kubectl get serviceaccount cert-manager -n cert-manager

# Check Vault role configuration
vault read auth/kubernetes/role/cert-manager

# Test token creation
kubectl create token cert-manager -n cert-manager --duration=1h
Key Benefits of This Modern Approach

No Manual Token Management: Uses Kubernetes native authentication
Automatic Certificate Rotation: cert-manager handles renewal automatically
Security Best Practices:

Uses service account tokens instead of static secrets
Implements private key rotation
Follows principle of least privilege


Production Ready: Includes monitoring, alerting, and proper security contexts
Maintenance Free: Minimal operational overhead once deployed

This setup provides enterprise-grade certificate management for free5GC OAuth2 implementation while following modern Kubernetes security practices and avoiding deprecated authentication methods.              