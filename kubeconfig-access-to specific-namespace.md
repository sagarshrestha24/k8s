# Create Kubernetes Service Accounts and Kubeconfigs

This document primarily uses kubectl and assumes you have access to permissions that can create and/or update these resources in your Kubernetes cluster:

  1. Kubernetes Service Account(s)
  2. Kubernetes Roles and Rolebindings
  3. (Optionally) Kubernetes ClusterRoles and Rolebindings

## Create the Kubernetes Service Account

You can use the following manifest to create a service account. Replace NAMESPACE with the namespace you want to use and, optionally, rename the service account.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: limited-service-account
  namespace: NAMESPACE
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
 name: sa-secret
 annotations:
 kubernetes.io/service-account.name: “limited-service-account”
```
Then create the object:
```
kubectl apply -f spinnaker-service-account.yml
```

Grant namespace-specific permissions

If you only want the service account to access specific namespaces, you can create a role with a set of permissions and rolebinding to attach the role to the service account. You can do this multiple times. Additionally, you will have to explicitly do this for the namespace where the service account is, as it is not implicit.

Important points:

  A Kubernetes Role exists in a given namespace and grants access to items in that namespace
  A Kubernetes RoleBinding exists in a given namespace and attaches a role in that namespace to some principal (in this case, a service account). The principal (service account) may be in another namespace.
  If you have a service account in namespace source and want to grant access to namespace target, then do the following:
        1. Create the service account in namespace source
        2. Create a Role in namespace target
        3. Create a RoleBinding in namespace target, with the following properties:
            RoleRef pointing to the Role (that is in the same namespace target)
            Subject pointing to the service account and namespace where the service account lives (in namespace source)

Change the names of resources to match your environment, as long as the namespaces are correct, and the subject name and namespace match the name and namespace of your service account.

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: spinnaker-role
  namespace: target # Should be namespace you are granting access to
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spinnaker-rolebinding
  namespace: target # Should be namespace you are granting access to
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: spinnaker-role # Should match name of Role
subjects:
- namespace: source # Should match namespace where SA lives
  kind: ServiceAccount
  name: spinnaker-service-account # Should match service account name, above
```
Then, create the object:

```
kubectl apply -f spinnaker-role-and-rolebinding-target.yml
```

Get the service account and token

Run these commands (or commands like these) to get the token for your service account and create a kubeconfig with access to the service account.

This file will contain credentials for your Kubernetes cluster and should be stored securely.

```
# Update these to match your environment
SERVICE_ACCOUNT_NAME=spinnaker-service-account
CONTEXT=$(kubectl config current-context)
NAMESPACE=spinnaker

NEW_CONTEXT=spinnaker
KUBECONFIG_FILE="kubeconfig-sa"


SECRET_NAME=$(kubectl get serviceaccount ${SERVICE_ACCOUNT_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.secrets[0].name}')
TOKEN_DATA=$(kubectl get secret ${SECRET_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.data.token}')

TOKEN=$(echo ${TOKEN_DATA} | base64 -d)

# Create dedicated kubeconfig
# Create a full copy
kubectl config view --raw > ${KUBECONFIG_FILE}.full.tmp
# Switch working context to correct context
kubectl --kubeconfig ${KUBECONFIG_FILE}.full.tmp config use-context ${CONTEXT}
# Minify
kubectl --kubeconfig ${KUBECONFIG_FILE}.full.tmp \
  config view --flatten --minify > ${KUBECONFIG_FILE}.tmp
# Rename context
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  rename-context ${CONTEXT} ${NEW_CONTEXT}
# Create token user
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-credentials ${CONTEXT}-${NAMESPACE}-token-user \
  --token ${TOKEN}
# Set context to use token user
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-context ${NEW_CONTEXT} --user ${CONTEXT}-${NAMESPACE}-token-user
# Set context to correct namespace
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-context ${NEW_CONTEXT} --namespace ${NAMESPACE}
# Flatten/minify kubeconfig
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  view --flatten --minify > ${KUBECONFIG_FILE}
# Remove tmp
rm ${KUBECONFIG_FILE}.full.tmp
rm ${KUBECONFIG_FILE}.tmp
```
You should end up with a kubeconfig that can access your Kubernetes cluster with the desired target namespaces.
