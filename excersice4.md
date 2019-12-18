
#Featch the file https://raw.githubusercontent.com/Azure/kubernetes-keyvault-flexvol/master/deployment/kv-flexvol-installer.yaml

```
wget https://raw.githubusercontent.com/Azure/kubernetes-keyvault-flexvol/master/deployment/kv-flexvol-installer.yaml
```

# Deploy the FlexVol into the proper namespace
```
kubectl create -f .\flexkv.yaml --namespace=api-dev
```

# Validate it's there
```
kubectl get pods -n kv --namespace=api-dev
```

# Create a new SPN
```
az ad sp create-for-rbac --name myServicePrincipalName 
```

# Define variables with the values shown on the previous step
```
$KV_NAME = "hackerkey"
$ResourceGroupName = "teamResources"
$var_azurerm_sub_id= "afbc1e5d-9b9e-4846-bfb4-8d46b8c87231"
$SpnClient = "APP_ID"
$SpnPassword = "SPN_PASSWORD"

# Create the KeyVault if not present
```
az keyvault create --name  $KV_NAME --resource-group $ResourceGroupName
```

# Add read permissions to the SPN
```
az role assignment create --role Reader --assignee $SpnClient --scope /subscriptions/$var_azurerm_sub_id/resourcegroups/$ResourceGroupName/providers/Microsoft.KeyVault/vaults/$KV_NAME 
```

# Assign key vault permissions to your service principal
```
az keyvault set-policy -n $KV_NAME --key-permissions get --spn $SpnClient
az keyvault set-policy -n $KV_NAME --secret-permissions get --spn $SpnClient
az keyvault set-policy -n $KV_NAME --certificate-permissions get --spn $SpnClient
```

# Populate the KV with the required secrets
Create secrets called with their respective values:
1. sqldbname
1. sqlpassword
1. sqlServer
1. sqluser

# Create Generic Credentials within the Kubernetes Cluster
```
kubectl create secret generic kvcreds --from-literal clientid=$SpnClient --from-literal clientsecret=$SpnPassword --type=azure/kv --namespace=api-dev
kubectl get secret --namespace=api-dev
```

# UPpdate your apps' deployments to use FlexVol

## POI
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poi
  labels:
    app: poi
  namespace: api-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: poi
  template:
    metadata:
      labels:
        app: poi
    spec:
      containers:
      - name: poi
        image: registryncl1796.azurecr.io/tripinsights/poi/v1
        env:
          - name: ASPNETCORE_ENVIRONMENT
            value: local
        ports:
        - containerPort: 80
        volumeMounts:
        - name: secrets
          mountPath: /secrets
          readOnly: true
      volumes:
      - name: secrets
        flexVolume:
          driver: "azure/kv"
          secretRef:
            name: kvcreds                                          # [OPTIONAL] not required if using Pod Identity
          options:                                                 
            usepodidentity: "false"                                # [OPTIONAL] if not provided, will default to "false"
            usevmmanagedidentity: "false"                          # [OPTIONAL new in version >= v0.0.15] if not provided, will default to "false"
            vmmanagedidentityclientid: "clientid"                  # [OPTIONAL new in version >= v0.0.15] use the client id to specify which user assigned managed identity to use, leave empty to use system assigned managed identity
            keyvaultname: "hackerkey"                      # [REQUIRED] the name of the KeyVault
            keyvaultobjectnames: "sqluser;sqlpassword;sqlServer;sqldbname"             # [REQUIRED] list of KeyVault object names (semi-colon separated)
            keyvaultobjectaliases: "SQL_USER;SQL_PASSWORD;SQL_SERVER;SQL_DBNAME"         #secret.json"      # [OPTIONAL] list of KeyVault object aliases
            keyvaultobjecttypes: "secret;secret;secret;secret"                   # [REQUIRED] list of KeyVault object types: secret, key, cert
            keyvaultobjectversions: ""                             # [OPTIONAL] list of KeyVault object versions (semi-colon separated), will get latest if empty
            resourcegroup: "teamResources"                     # [REQUIRED for version < v0.0.14] the resource group of the KeyVault
            subscriptionid: "afbc1e5d-9b9e-4846-bfb4-8d46b8c87231" # [REQUIRED for version < v0.0.14] the subscription ID of the KeyVault
            tenantid: "d4359d99-bcd2-4b4f-85a1-b47245e38edd"       # [REQUIRED] the tenant ID of the KeyVault
---  
apiVersion: v1
kind: Service
metadata:
  name: poi
  namespace: api-dev
spec:
  type: LoadBalancer
  selector:
    app: poi
  ports:
  - protocol: TCP
    targetPort: 80
    port: 8080
```