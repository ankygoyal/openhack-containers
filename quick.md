# Quick Steps Guide

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
  -e SQLDB=mydrivingDB `
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
