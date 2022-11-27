# AKS-Workload-identity-in-steps
AKS-Workload-identity-in-steps




(1) Create a resource group
========================================
az group create --name myResourceGroup1 --location eastus


(2) Install the aks-preview Azure CLI extension
========================================
az extension add --name aks-preview
az extension update --name aks-preview


(3) Create AKS cluster
========================================
az aks create -g myResourceGroup1 -n myAKSCluster1 --node-count 1 --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys

(4) Export environmental variables
========================================
export AKS_OIDC_ISSUER="$(az aks show -n myAKSCluster1 -g myResourceGroup1 --query "oidcIssuerProfile.issuerUrl" -otsv)"

# environment variables for the Azure Key Vault resource
export KEYVAULT_NAME="azwi-kv-tutorial1"
export KEYVAULT_SECRET_NAME="my-secret1"
export RESOURCE_GROUP="myResourceGroup1"
export LOCATION="westcentralus"

# environment variables for the Kubernetes Service account & federated identity credential
export SERVICE_ACCOUNT_NAMESPACE="default"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"

# environment variables for the Federated Identity
export SUBSCRIPTION="38e1b8c4-c5bc-4dd5-a7e0-e909b45f4fad"
# user assigned identity name
export UAID="fic-test-ua"
# federated identity name
export FICID="fic-test-fic-name"

(5) Create an Azure Key Vault and secret
========================================
az keyvault create --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --name "${KEYVAULT_NAME}"
az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name "${KEYVAULT_SECRET_NAME}" --value 'Hello22!'

(6) Create a managed identity and grant permissions to access the secret
========================================
az account set --subscription "${SUBSCRIPTION}"
az identity create --name "${UAID}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --subscription "${SUBSCRIPTION}"


export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${UAID}" --query 'clientId' -otsv)"

az keyvault set-policy --name "${KEYVAULT_NAME}" --secret-permissions get --spn "${USER_ASSIGNED_CLIENT_ID}"

(7) Create Kubernetes service account
========================================
az aks get-credentials -n myAKSCluster1 -g myResourceGroup1

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF

(8) Establish federated identity credential
========================================
az identity federated-credential create --name ${FICID} --identity-name ${UAID} --resource-group ${RESOURCE_GROUP} --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}

(9) Deploy the workload
========================================
export KEYVAULT_URL="$(az keyvault show -g ${RESOURCE_GROUP} -n ${KEYVAULT_NAME} --query properties.vaultUri -o tsv)"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: quick-start1
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  containers:
    - image: ghcr.io/azure/azure-workload-identity/msal-go
      name: oidc
      env:
      - name: KEYVAULT_URL
        value: ${KEYVAULT_URL}
      - name: SECRET_NAME
        value: ${KEYVAULT_SECRET_NAME}
  nodeSelector:
    kubernetes.io/os: linux
EOF

kubectl describe pod quick-start
kubectl logs quick-start

I1013 22:49:29.872708       1 main.go:30] "successfully got secret" secret="Hello!"

(10) Clean up resources
========================================
kubectl delete pod quick-start
kubectl delete sa "${SERVICE_ACCOUNT_NAME}" --namespace "${SERVICE_ACCOUNT_NAMESPACE}"
az group delete --name "${RESOURCE_GROUP}"
