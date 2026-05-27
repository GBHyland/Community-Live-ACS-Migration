# Migrate OnPrem to Cloud
---  

> While running this guide it is optimal to have two terminal tabs: 
> - Docker Compose Tab: You'll run the stage docker commands here
> - Testing Tab: You'll run test commands here to test services stood up by the docker compose files.
> - Pay attention to the commands below to use the right tabs: **Docker Command** and **Test**
> - When progressing to the next stage, use CTRL+C in the **Docker** tab to stop the services and the 

---

## Stage 01
Start a small ACS environment named acs26-stage01 with two services: PostgreSQL and Alfresco Content Repository

### Docker Commands:
```
docker compose --env-file .env -f stages/01-repo/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/01-repo/compose.yaml exec -T postgres \
  sh -c 'pg_isready -d "$POSTGRES_DB" -U "$POSTGRES_USER"'
```
> Expected:
> /var/run/postgresql:5432 - accepting connections

  
Validate the Repository
```
curl -f http://localhost:8080/alfresco/api/-default-/public/alfresco/versions/1/probes/-ready-
```
> Expected:
> {"entry":{"message":"readyProbe: Success - Tested"}}


---  

## Stage 02
Keep the same services stage01, and add Transform Core AIO so Alfresco can perform document transformations.

### Docker Commands:
```
docker compose --env-file .env -f stages/01-repo/compose.yaml down
```
```
docker compose --env-file .env -f stages/02-transform-core-aio/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/01-repo/compose.yaml exec -T postgres \
  sh -c 'pg_isready -d "$POSTGRES_DB" -U "$POSTGRES_USER"'
```
> Expected:
> /var/run/postgresql:5432 - accepting connections

  
Validate the Repository
```
curl -f http://localhost:8080/alfresco/api/-default-/public/alfresco/versions/1/probes/-ready-
```
> Expected:
> {"entry":{"message":"readyProbe: Success - Tested"}}


> [!NOTE]
> New Tests for this stage:  

Validate Transform
```
docker compose --env-file .env -f stages/02-transform-core-aio/compose.yaml exec -T transform-core-aio \
  curl -sf http://localhost:8090/ready
```
> Expected:
> Success - No transform.

---  












