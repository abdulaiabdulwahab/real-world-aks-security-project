# real-world-aks-security-project

# Zero-Trust AKS Security

## Project Overview

This project demonstrates how to secure an Azure Kubernetes Service (AKS) workload using a zero-trust approach.

The goal is to allow an application running in AKS to securely access Azure Key Vault without storing usernames, passwords, service principal secrets, or other credentials inside the application or Kubernetes manifests.

The project uses Microsoft Entra Workload Identity, Azure Key Vault, Pod Security Admission, Cilium Network Policies, and Azure RBAC.

## Technologies Used

- Azure Kubernetes Service (AKS)
- Azure CLI
- Kubernetes
- Microsoft Entra ID
- Azure RBAC
- Microsoft Entra Workload Identity
- OpenID Connect (OIDC)
- Azure Managed Identity
- Azure Key Vault
- Secrets Store CSI Driver
- Cilium
- Kubernetes Network Policies
- Pod Security Admission

## Security Features Implemented

This project includes:

- Microsoft Entra ID authentication
- Azure RBAC for AKS access
- Disabled local administrator accounts
- API server IP restrictions
- Microsoft Entra Workload Identity
- OIDC federation
- User-assigned managed identity
- Azure Key Vault integration
- Least-privilege Key Vault access
- Pod Security Admission
- Secure container security contexts
- Default-deny Kubernetes Network Policies
- Automatic node security patching
- AKS Image Cleaner

## Architecture

```text
Developer
    |
    v
Microsoft Entra ID
    |
    v
Azure RBAC
    |
    v
AKS Cluster
    |
    +-- Pod Security Admission
    |
    +-- Cilium Network Policies
    |
    +-- Workload Identity
              |
              v
       Managed Identity
              |
              v
        Azure Key Vault
```

## Create the Resource Group

```bash
export RG2="rg-aks-zero-trust"
export LOCATION="canadacentral"
export AKS2="aks-zero-trust"

az group create \
  --name "$RG2" \
  --location "$LOCATION"
```

## Create the AKS Cluster

```bash
az aks create \
  --resource-group "$RG2" \
  --name "$AKS2" \
  --location "$LOCATION" \
  --node-count 1 \
  --enable-aad \
  --enable-azure-rbac \
  --disable-local-accounts \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --enable-addons azure-keyvault-secrets-provider \
  --node-os-upgrade-channel SecurityPatch \
  --enable-image-cleaner \
  --image-cleaner-interval-hours 24 \
  --generate-ssh-keys
```

## Connect to AKS

```bash
az aks get-credentials \
  --resource-group "$RG2" \
  --name "$AKS2" \
  --overwrite-existing
```

Verify:

```bash
kubectl get nodes
```

## Create Azure Key Vault

```bash
export KV_NAME="kvakssec$RANDOM$RANDOM"

az keyvault create \
  --name "$KV_NAME" \
  --resource-group "$RG2" \
  --location "$LOCATION" \
  --enable-rbac-authorization true
```

A test secret was then stored in Key Vault.

```bash
az keyvault secret set \
  --vault-name "$KV_NAME" \
  --name "db-password" \
  --value "ThisIsOnlyALabPassword123!"
```

Real credentials should never be committed to source control.

## Create a Managed Identity

```bash
export IDENTITY_NAME="mi-aks-prod-app"

az identity create \
  --name "$IDENTITY_NAME" \
  --resource-group "$RG2" \
  --location "$LOCATION"
```

The managed identity allows the AKS workload to authenticate to Azure without storing credentials.

## Grant Least-Privilege Access

The managed identity was assigned the:

```text
Key Vault Secrets User
```

role.

This allows the application to read Key Vault secrets without giving it permission to create, modify, or delete them.

## Configure Workload Identity

The AKS cluster uses an OIDC issuer to establish trust between a Kubernetes ServiceAccount and Microsoft Entra ID.

The authentication flow is:

```text
Kubernetes Pod
      |
      v
ServiceAccount
      |
      v
OIDC Token
      |
      v
Microsoft Entra ID
      |
      v
Managed Identity
      |
      v
Azure Key Vault
```

This removes the need to store Azure credentials inside the application.

## Create the Secure Namespace

```bash
kubectl create namespace prod-app
```

Enable the restricted Pod Security Standard:

```bash
kubectl label --overwrite namespace prod-app \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

## Secure the Workload

The application uses a secure container configuration including:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000

  seccompProfile:
    type: RuntimeDefault
```

Container-level protections include:

```yaml
securityContext:
  allowPrivilegeEscalation: false

  capabilities:
    drop:
      - ALL

  readOnlyRootFilesystem: true
```

These controls reduce the privileges available to the container.

## Azure Key Vault Integration

The Secrets Store CSI Driver mounts the Key Vault secret directly into the pod.

Example mount location:

```text
/mnt/secrets-store/db-password
```

The application can therefore use the secret without storing it in the container image or source code.

The secret can be verified without displaying its value:

```bash
kubectl exec \
  -n prod-app \
  secure-prod-app \
  -- sh -c 'test -s /mnt/secrets-store/db-password && echo "Secret exists and is not empty."'
```

## Network Security

Default-deny Network Policies were implemented.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny-ingress
  namespace: prod-app

spec:
  podSelector: {}

  policyTypes:
    - Ingress
```

A separate policy was also used to deny egress traffic by default.

The security model becomes:

```text
Traffic
   |
   v
Default Deny
   |
   +---- Explicit Allow Rule ----> Allowed
   |
   +---- No Allow Rule ----------> Blocked
```

## Security Validation

Useful commands include:

```bash
kubectl get pods -n prod-app
```

```bash
kubectl get networkpolicy -n prod-app
```

```bash
kubectl get namespace prod-app --show-labels
```

Verify Workload Identity:

```bash
az aks show \
  --resource-group "$RG2" \
  --name "$AKS2" \
  --query securityProfile.workloadIdentity
```

Verify OIDC:

```bash
az aks show \
  --resource-group "$RG2" \
  --name "$AKS2" \
  --query oidcIssuerProfile
```

Verify local accounts are disabled:

```bash
az aks show \
  --resource-group "$RG2" \
  --name "$AKS2" \
  --query disableLocalAccounts
```

## Troubleshooting

### Key Vault Returns 403 Forbidden

Check the managed identity's role assignment:

```bash
az role assignment list \
  --assignee "$PRINCIPAL_ID" \
  --scope "$KV_ID" \
  --output table
```

The identity should have:

```text
Key Vault Secrets User
```

### Secret Fails to Mount

Check the pod:

```bash
kubectl describe pod secure-prod-app \
  -n prod-app
```

Look for errors such as:

```text
FailedMount
SecretProviderClass
Forbidden
```

Check the SecretProviderClass:

```bash
kubectl get secretproviderclass \
  -n prod-app
```

### Workload Identity Authentication Fails

Check the Kubernetes ServiceAccount:

```bash
kubectl describe serviceaccount \
  prod-app-sa \
  -n prod-app
```

Verify the pod contains the Workload Identity label:

```bash
kubectl get pod secure-prod-app \
  -n prod-app \
  --show-labels
```

The pod should contain:

```text
azure.workload.identity/use=true
```

Also verify the federated identity credential:

```bash
az identity federated-credential list \
  --identity-name "$IDENTITY_NAME" \
  --resource-group "$RG2" \
  --output table
```

### Network Connectivity Is Blocked

Check the active policies:

```bash
kubectl get networkpolicy \
  -n prod-app
```

Inspect them:

```bash
kubectl describe networkpolicy \
  -n prod-app
```

With default-deny enabled, communication will fail unless an explicit allow rule exists.

## Skills Practiced

This project provided hands-on experience with:

- AKS Security
- Zero-Trust Architecture
- Microsoft Entra ID
- Azure RBAC
- Microsoft Entra Workload Identity
- OIDC Federation
- Managed Identities
- Azure Key Vault
- Secrets Store CSI Driver
- Kubernetes ServiceAccounts
- Pod Security Admission
- Kubernetes Security Contexts
- Cilium
- Kubernetes Network Policies
- Least-Privilege Access
- Secret Management
- Container Security

## Cleanup

Delete all project resources when finished:

```bash
az group delete \
  --name "$RG2" \
  --yes \
  --no-wait
```

## Conclusion

This project demonstrates how AKS workloads can securely access Azure services without storing static credentials.

By combining Microsoft Entra Workload Identity, Azure Key Vault, Azure RBAC, Pod Security Admission, secure container configurations, and Cilium Network Policies, the application follows a stronger zero-trust security model.

The project also demonstrates how identity, secrets management, workload isolation, and network security can be combined to protect production-style Kubernetes workloads in Azure.