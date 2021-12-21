# akv2k8s
basic implementation


## Prequisites
### Prerequisites needed to complete the lab:

## Azure Resources:

1. Azure Resource Group: `az group create -l westeurope -n akv2k8s-test `
2. Azure Key Vault: `az keyvault create -n akv2k8s-test -g akv2k8s-test `
3. Add secrets (required for the secrets-lab):
`az keyvault secret set --vault-name akv2k8s-test --name my-secret --value "My super secret" ` </br>
`az keyvault secret set --vault-name akv2k8s-test --name my-other-secret --value "My other super secret"`
4. Authorize the secrets: `az keyvault set-policy -n akv2k8s-test --spn <spn for akv2k8s> --secret-permissions get`
5. Add certificate: `az keyvault certificate create --vault-name akv2k8s-test --name my-certificate -p "$(az keyvault certificate get-default-policy -o json)"`
6. Authorize access to Certificates: `az keyvault set-policy -n akv2k8s-test --spn <spn for akv2k8s> --certificate-permissions get`
7. Add Signing key: `az keyvault key create --vault-name akv2k8s-test --name my-key`
8. Authorize Access to Keys: `az keyvault set-policy -n akv2k8s-test --spn <spn for akv2k8s> --key-permissions get`
9. Kubernetes resources: Create namespace:
```yml
apiVersion: v1
kind: Namespace
metadata:
  name: akv-test
  labels:
    azure-key-vault-env-injection: enabled
```
10. Apply configuration: `kubectl apply -f namespace.yaml`

### Installation:

1. Create a dedicated namespace: `kubectl create ns akv2k8s`
2. Install with Helm on Azure AKS:
```
helm repo add spv-charts https://charts.spvapi.no
helm repo update
```
```
helm upgrade --install akv2k8s spv-charts/akv2k8s \
  --namespace akv2k8s
```

## Inject Secret
### Inject an Azure Key Vault secret directly into a container application

1. Create a definition for the Azure Key Vault secret we want to inject:
```yml
apiVersion: spv.no/v2beta1
kind: AzureKeyVaultSecret
metadata:
  name: secret-inject 
  namespace: akv-test
spec:
  vault:
    name: akv2k8s-test # name of key vault
    object:
      name: my-secret # name of the akv object
      type: secret # akv object type
```
2. Apply to Kubernetes: `$ kubectl apply -f akvs-secret-inject.yaml`
3. Check List AzureKeyVaultSecret's: `kubectl -n akv-test get akvs`
4. Deploy a Pod having a env-variable pointing to the secret above:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: akvs-secret-app
  namespace: akv-test
  labels:
    app: akvs-secret-app
spec:
  selector:
    matchLabels:
      app: akvs-secret-app
  template:
    metadata:
      labels:
        app: akvs-secret-app
    spec:
      containers:
      - name: akv2k8s-env-test
        image: spvest/akv2k8s-env-test:2.0.1
        args: ["TEST_SECRET"]
        env:
        - name: TEST_SECRET
          value: "secret-inject@azurekeyvault" # ref to akvs
```
5. Apply to KS8: `$ kubectl apply -f secret-deployment.yaml`
6. Check the log output from your Pod: `kubectl -n akv-test logs deployment/akvs-secret-app`
7. Cleanup: 
```bash
kubectl delete -f akvs-secret-inject.yaml
kubectl delete -f secret-deployment.yaml
```
* Things to note from the Deployment above:
```yml
containers:
  - name: akv2k8s-env-test

    image: spvest/akv2k8s-env-test:2.0.1 # 1.
    args: ["TEST_SECRET"] # 2.
    env:

    - name: TEST_SECRET # 3.
      value: "secret-inject@azurekeyvault" # 4.

```
1. We use a custom built Docker image for testing purposes that only outputs the content of the env-variables passed in as args in #2. Feel free to replace this with your own Docker image.
2. Again, specific for the Docker test image we are using (in #1), we pass in which environment variables we want the container to print values for
3. Name of the environment variable
4. By using the special akv2k8s Env Injector convention `<azure-key-vault-secret-name>@azurekeyvault` to reference the AzureKeyVaultSecret `secret-inject` we created earlier. The env-injector will download this secret from Azure Key Vault and inject into the executable running in your Container.
