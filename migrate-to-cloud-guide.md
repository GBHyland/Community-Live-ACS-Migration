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
docker compose --env-file .env -f stages/02-transform-core-aio/compose.yaml exec -T postgres \
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
> curl -sf http://localhost:8090/ready


---  

## Stage 03
The setup moves from simple transformations to a more complete Alfresco transform-service architecture, adding ActiveMQ, Shared File Store, and the Transform Router.

### Docker Commands:
```
docker compose --env-file .env -f stages/02-transform-core-aio/compose.yaml down
```
```
docker compose --env-file .env -f stages/03-transform-service-ats/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/03-transform-service-ats/compose.yaml exec -T postgres \
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
curl -f -u admin:admin http://localhost:8161
docker compose --env-file .env -f stages/03-transform-service-ats/compose.yaml exec -T shared-file-store \
  curl -sf http://localhost:8099/ready
docker compose --env-file .env -f stages/03-transform-service-ats/compose.yaml exec -T transform-router \
  curl -sf http://localhost:8095/actuator/health
```
> Expected:
> Ready Probe: Store 7ms{"groups":["liveness","readiness"],"status":"UP"}


---  

## Stage 04
The setup adds Search Services / Solr on top of the Stage 03 transform architecture. The new major service is solr6.

### Docker Commands:
```
docker compose --env-file .env -f stages/03-transform-service-ats/compose.yaml down
```
```
docker compose --env-file .env -f stages/04-solr-search-with-ats/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/04-solr-search-with-ats/compose.yaml exec -T postgres \
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

Validate Search
```
curl -H "X-Alfresco-Search-Secret: secret" \
  "http://localhost:8983/solr/alfresco/select?q=*&rows=1&wt=json"
curl -u "admin:admin" \
  -H "Content-Type: application/json" \
  -d '{"query":{"query":"test"},"paging":{"maxItems":1,"skipCount":0}}' \
  "http://localhost:8080/alfresco/api/-default-/public/search/versions/1/search"
```
> Expected something similar to:  
> { "responseHeader": {"status": 0, ...}, ... }  
> {"list": {"pagination": { "count": ..., ... },"entries": [ ... ]}}  


---  

## Stage 05
The setup builds a much more advanced ACS environment centered around OpenSearch-based indexing instead of Solr.

### Docker Commands:
```
docker compose --env-file .env -f stages/04-solr-search-with-ats/compose.yaml down
```
```
docker compose --env-file .env -f stages/05-opensearch-migration-with-ats/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/05-opensearch-migration-with-ats/compose.yaml exec -T postgres \
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

Validate Search (OpenSearch Migration)
```
curl -H "Content-Type: application/json" \
  -d '{"query":{"match_all":{}},"size":1}' \
  "http://localhost:9200/alfresco/_search"
curl -u "admin:admin" \
  -H "Content-Type: application/json" \
  -d '{"query":{"query":"test"},"paging":{"maxItems":1,"skipCount":0}}' \
  "http://localhost:8080/alfresco/api/-default-/public/search/versions/1/search"
```
> Expected:  
> { "hits": { "total": { "value": ..., ... }, "hits": [ ... ] } }  
> {"list": {"pagination": { "count": ..., ... },"entries": [ ... ]}}  


---  

## Stage 06
This stage turns the backend-focused ACS stack from Stage 05 into a much more complete, user-facing Alfresco platform environment, adding Digital Workspace (ADW), Control Center, Share and CORS support.

### Docker Commands:
```
docker compose --env-file .env -f stages/05-opensearch-migration-with-ats/compose.yaml down
```
```
docker compose --env-file .env -f stages/06-full-stack/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/06-full-stack/compose.yaml exec -T postgres \
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

1. Open `http://your-ip:8081/` (ADW) in your browser.
2. Log in with credentials from .env (loacted in root directory) (default should be: `admin` / `admin`).
3. Upload a new text document with a unique word in its body (for example: stage06-e2e-2026).
4. Search in ADW for that unique word and open the returned document.
5. Open `http://your-ip:8082/share` and confirm the same document appears.
6. Open `http://your-ip:8083/` (Control Center) and confirm login works.

> Expected:  
> Login works in all three UIs, upload succeeds, and full-text search returns the newly uploaded document.  
> This confirms repo, transform, messaging, and OpenSearch indexing are working together.  


---  

## Stage 07
This stage adds a centralized NGINX reverse proxy layer in front of all the Alfresco applications.

### Docker Commands:
```
docker compose --env-file .env -f stages/06-full-stack/compose.yaml down
```
```
docker compose --env-file .env -f stages/07-full-stack-proxy/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/07-full-stack-proxy/compose.yaml exec -T postgres \
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

Validate Proxy
```
docker compose --env-file .env -f stages/07-full-stack-proxy/compose.yaml ps proxy
curl -f http://localhost:8080/alfresco/api/-default-/public/alfresco/versions/1/probes/-ready-
curl -fL http://localhost:8080/workspace
curl -fL http://localhost:8080/share
```
> Expected:  
> proxy is Up and `/alfresco`, `/workspace`, and `/share` respond through port 8080


---  



















