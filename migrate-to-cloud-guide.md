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
> {"responseHeader":{"status":0,"QTime":547,"params":{"q":"*","rows":"1","wt":"json"}},"_original_parameters_":{"q":"*","defType":"afts","df":"suggest","rows":"1","wt":"json","echoParams":"explicit"},"_field_mappings_":{},"_date_mappings_":{},"_range_mappings_":{},"_pivot_mappings_":{},"_interval_mappings_":{},"_stats_field_mappings_":{},"_stats_facet_mappings_":{},"_facet_function_mappings_":{},"lastIndexedTx":18,"lastIndexedTxTime":1779892817918,"txRemaining":0,"response":{"numFound":780,"start":0,"docs":[{"DBID":844,"id":"_DEFAULT_!800000000000034c"}]},"processedDenies":false}
{"list":{"pagination":{"count":1,"hasMoreItems":true,"totalItems":6,"skipCount":0,"maxItems":1},"context":{"consistency":{"lastTxId":18}},"entries":[{"entry":{"isFile":true,"createdByUser":{"id":"System","displayName":"System"},"modifiedAt":"2026-05-27T14:40:01.802+0000","nodeType":"cm:content","content":{"mimeType":"application/x-javascript","mimeTypeName":"JavaScript","sizeInBytes":2271,"encoding":"UTF-8"},"parentId":"e5701d87-adc1-4ba3-b01d-87adc11ba33c","createdAt":"2026-05-27T14:40:01.802+0000","isFolder":false,"search":{"score":1.0},"modifiedByUser":{"id":"System","displayName":"System"},"name":"example test script.js.sample","location":"nodes","id":"941c4ecc-26ad-4be2-9c4e-cc26ad8be202"}}]}}
















