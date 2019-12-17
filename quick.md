# Excersice 1

On a privileged PS ISE session run:

## Setup variables
```
$GIT_REPO_DIR = "~/openhack-containers"
$SQL_DB_NAME = "mydrivingDB"
$SQL_USER = "SA"
$SQL_PASSWORD = "sqladminnCl1796@zC8yn3Hf7"
```

## Create the DB instance
```
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=$SQL_PASSWORD" `
  -p 1433:1433 --name $SQL_DB_NAME `
  -d mcr.microsoft.com/mssql/server:2017-latest
```
​
## List the DB's IP address
```
docker network inspect bridge
```

## Define the DB IP adreess variable from the information gathered above
```
$SQL_SERVER = "172.17.0.2"
```

## Create the DB on the (To be validated)
```
docker exec $SQL_DB_NAME /opt/mssql-tools/bin/sqlcmd `
  -S localhost `
  -U $SQL_USER `
  -P $SQL_PASSWORD `
  -Q "CREATE DATABASE $SQL_DB_NAME"
```

## We build the POI Image
```
cd ${GIT_REPO_DIR}/src/poi
docker build -f ../../dockerfiles/Dockerfile_3 `
  -t "tripinsights/poi:1.0" `
  .
```
​
## Run the POI container
```
docker run -d -p 8080:80 `
  --name poi `
  -e "SQL_PASSWORD=$SQL_PASSWORD" `
  -e "SQL_SERVER=$SQL_SERVER" `
  -e "ASPNETCORE_ENVIRONMENT=Local" `
  tripinsights/poi:1.0
```

## Validate that the POI Container is running
```
curl -i -X GET 'http://localhost:8080/api/poi/healthcheck' 
```

## We perform the data upload
```
docker run --network bridge `
  -e SQLFQDN=$SQL_SERVER `
  -e SQLUSER=$SQL_USER `
  -e SQLPASS=$SQL_PASSWORD `
  -e SQLDB=$SQL_DB_NAME `
  openhack/data-load:v1
```

## Check the POI End point URL
```
 'http://localhost:8080/api/poi/264ffaa3-1fe8-4fb0-a4fb-63bdbc9999ae'
 ```

## Build the other services
### User API
```
cd ${GIT_REPO_DIR}/src/user-java
docker build -f ../../dockerfiles/Dockerfile_0 `
  -t "tripinsights/user-java:1.0" `
  .
```

### User API Profile
```
cd ${GIT_REPO_DIR}/src/userprofile
docker build -f ../../dockerfiles/Dockerfile_2 `
  -t "tripinsights/userprofile:1.0" `
  .
```

### User Trip Viewer
```
cd ${GIT_REPO_DIR}/src/tripviewer
docker build -f ../../dockerfiles/Dockerfile_1 `
  -t "tripinsights/tripviewer:1.0" `
  .
```

### Trips API
```
cd ${GIT_REPO_DIR}/src/trips
docker build -f ../../dockerfiles/Dockerfile_4 `
  -t "tripinsights/trips:1.0" `
  .
```

# Push the images to the ACR

## Setup variables
```
$ACR_NAME = "registryncl1796"
$ACR_USER = "registryncl1796"
$ACR_PASSWORD = "u9vBIotELDhDA+egeYTkIFWnfG2gQq8u"  
$ACR_SUFFIX = "registryncl1796.azurecr.io"
$ACR_
```


## Login to the ACR
```
az login
```

## Tag the Images
```
docker tag tripinsights/poi:1.0 registryncl1796.azurecr.io/tripinsights/poi:1.0
docker tag tripinsights/trips:1.0 registryncl1796.azurecr.io/tripinsights/trips:1.0
docker tag tripinsights/tripviewer:1.0 registryncl1796.azurecr.io/tripinsights/tripviewer:1.0
docker tag tripinsights/userprofile:1.0 registryncl1796.azurecr.io/tripinsights/userprofile:1.0
docker tag tripinsights/user-java:1.0 registryncl1796.azurecr.io/tripinsights/user-java:1.0
```

## Push the Images
```
docker push registryncl1796.azurecr.io/tripinsights/poi/v1
docker push registryncl1796.azurecr.io/tripinsights/trips/v1
docker push registryncl1796.azurecr.io/tripinsights/tripviewer/v1
docker push registryncl1796.azurecr.io/tripinsights/userprofile/v1
docker push registryncl1796.azurecr.io/tripinsights/user-java/v1
```

# Excersice 2
## Enable AKS to pull images from ACR

```
az aks update -n HumongousInsurance -g teamResources --attach-acr registryncl1796
```

# Get Cluster credentials
```
az aks get-credentials --resource-group teamResources --name HumongousInsurance
```

# Validate Credentials and get nodes
```
kubectl get nodes
```

## POI Yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poi
  labels:
    app: poi
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
          - name: SQL_PASSWORD
            value: zC8yn3Hf7
          - name: SQL_USER
            value: sqladminnCl1796
          - name: ASPNETCORE_ENVIRONMENT
            value: Production
          - name: SQL_SERVER
            value: sqlserverncl1796.database.windows.net
        ports:
        - containerPort: 80
---  
apiVersion: v1
kind: Service
metadata:
  name: poi
spec:
  type: LoadBalancer
  selector:
    app: poi
  ports:
  - protocol: TCP
    targetPort: 80
    port: 8080
```

## Deploy the POI pods
Save the above yml file into a file

```
kubectl apply -f azure-vote.yaml
```
