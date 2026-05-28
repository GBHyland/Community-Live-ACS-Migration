# Migrate OnPrem to Cloud
---  

## Exercises in this Lab
| Stage | Formal Title                                    |
| ----- | ----------------------------------------------- |
| 01    | Foundational ACS Repository Deployment          |
| 02    | Integrated Content Transformation Services      |
| 03    | Distributed Transformation Service Architecture |
| 04    | Enterprise Search Services Integration          |
| 05    | OpenSearch and Live Indexing Architecture       |
| 06    | User Interface and Application Access Layer     |
| 07    | Centralized Reverse Proxy and Routing Layer     |
| 08    | Production-Ready Operational Architecture       |
| 09    | Custom Addon and Extension Deployment Platform  |
| 10    | Externalized Configuration and Data Management  |
| 11    | Secure HTTPS and SSL Access Architecture        |


---  


> While running this guide it is optimal to have two terminal tabs: 
> - Docker Compose Tab: You'll run the stage docker commands here
> - Testing Tab: You'll run test commands here to test services stood up by the docker compose files.
> - Pay attention to the commands below to use the right tabs: **Docker Command** and **Test**
> - When progressing to the next stage, use CTRL+C in the **Docker** tab to stop the services and the 

---  

## Setup & Discovery  
1. In Visual Studio Code, open a folder location to the planned installation directory, then open a terminal window within VS Code.
2. Clone this github:
```
git clone https://github.com/GBHyland/Community-Live-ACS-Migration.git
```

3. CD into the `Community-Live-ACS-Migration` directory and dedicate this terminal tab to your Docker Compose commands (suggest naming this tab `DOCKER`).
```
cd Community-Live-ACS-Migration
```

4. Open the `.env` file and change the `NGINX_SERVER_NAME=localhost` value, replacing `localhost` with your server's IP adress. ex:
```
NGINX_SERVER_NAME=3.85.22.172
```  
 
6. Open a new Terminal Tab and navigate to the same `Community-Live-ACS-Migration` directory; dedicate this tab to testing commands (suggest naming this tab `TESTING`).
7. In your testing terminal, use cat to open the .env file and review the content:
```
cat .env
```

> Note the different variables and attributes that will be used by the Docker Compose files in each Stage later.

8. Log in with Quay.io: (you will need your Quay username and password)
```
# Use your quay username and password/token
docker login quay.io
```


---  

## Stage 01
Start a small ACS environment named acs26-stage01 with two services: PostgreSQL and Alfresco Content Repository

### Docker Commands:
```
docker compose --env-file .env -f stages/01-repo/compose.yaml up
```

### Test:

---  
> NOTE:
> For the first test (and here on) ensure you are testing in your second terminal tab.
> Also, ensure you are testing from the `Community-Live-ACS-Migration` directory.
> If you get a `permission denied while trying to connect to the docker API` error, run these commands in your testing terminal tab:
```
sudo usermod -aG docker ubuntu
```
```
newgrp docker
```  
---  

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
> Ready Probe: Success - Transform


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
3. Open `http://your-ip:8082/share` (use same credentials from above).
4. Create a site then create a new text document with a unique word in its body (for example: `stage06-e2e-2026`).
5. Search in share for that unique word and open the returned document.
7. Open `http://your-ip:8083/` (Control Center) and confirm login works.

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

**Test in the browser**
1. Open `http://your-ip:8080/share` in your browser.
2. Log in with credentials from .env (loacted in root directory) (default should be: `admin` / `admin`).
> Expected:  
> Alfresco share should load via port 8080 (proxy)


---  

## Stage 08
This stage essentially transforms the Stage 07 environment into a much more production-ready, resilient, and operationally managed ACS platform, focusing on Reliability, Persistence, Resource management, Health monitoring, Operational stability, and Container lifecycle management.

### Docker Commands:
```
docker compose --env-file .env -f stages/07-full-stack-proxy/compose.yaml down
```
```
docker compose --env-file .env -f stages/08-best-practices/compose.yaml up
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/08-best-practices/compose.yaml exec -T postgres \
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

Validate Runtime Controls
```
docker compose --env-file .env -f stages/08-best-practices/compose.yaml ps
docker compose --env-file .env -f stages/08-best-practices/compose.yaml ps | grep -E "healthy|running"
```
> Expected:  
> core services report healthy/running and compose config contains deploy/resources/healthcheck/restart/service_healthy directives


---  

## Stage 09
This stage builds on Stage 08 by adding custom addon support for Alfresco Repository and Share, changing the Alfresco Repository and Share to add a customization layer making it more extensible.

### Docker Commands:
Fetch artifacts automatically:
```
cd stages/09-addons
../../shared/fetch-addons.sh
```
```
cd ../..  
```
```
docker compose --env-file .env -f stages/08-best-practices/compose.yaml down
```
```
docker compose --env-file .env -f stages/09-addons/compose.yaml up --build
```

### Test:
Validate the Database  
```
docker compose --env-file .env -f stages/09-addons/compose.yaml exec -T postgres \
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

Validate Addond Build:
```
docker compose --env-file .env -f stages/09-addons/compose.yaml ps alfresco share
docker image ls --format '{{.Repository}}:{{.Tag}}' | grep -E \
  'local/alfresco-content-repository-addons|local/alfresco-share-addons'
```
> Expected:  
> alfresco/share are Up and local addon images are present


---  

## Stage 10
This stage builds directly on Stage 09 by introducing a much more developer-friendly and externally managed configuration approach by moving important Alfresco data and configuration OUTSIDE the container images.

### Docker Commands:
```
docker compose --env-file .env -f stages/09-addons/compose.yaml down
```
```
docker compose --env-file .env -f stages/10-restore-onprem/compose.yaml up --build
```

### Test:
> [!NOTE]
> New Tests for this stage:  

Validate Restore Inputs:
```
docker compose --env-file .env -f stages/10-restore-onprem/compose.yaml exec -T alfresco sh -c \
  'test -d /usr/local/tomcat/alf_data && test -d /usr/local/tomcat/shared/classes/alfresco/extension && echo "restore mounts present"'
test -s shared/reindex/reindex.prefixes-file.json && echo "prefix map generated"
docker compose --env-file .env -f stages/10-restore-onprem/compose.yaml ps search-reindexing search-live-indexing
```
> Expected:  
> restore mounts are present, prefix map file exists, reindexing runs/completes, and live indexing starts afterwards


---  

## Stage 11
This stage is the introduction of HTTPS / SSL-secured platform access through the NGINX proxy, adding SSL/TLS certificates, HTTPS routing, Secure browser access, Secure Share/ADW URLs, and Secure proxy configuration.

### Docker Commands:
Replace the IP address with your server IP!
```
cd stages/11-security-local
NGINX_SERVER_NAME=your-ip ./generate-certs.sh
```
```
cd ../..
```
```
docker compose --env-file .env -f stages/10-restore-onprem/compose.yaml down
```
```
docker compose --env-file .env -f stages/11-security-local/compose.yaml up --build
```

### Test:
> [!NOTE]
> New Tests for this stage:  

Validate TLS Security:
```
curl -k -f https://localhost:8443/alfresco/api/-default-/public/alfresco/versions/1/probes/-ready-
curl -I http://localhost:8080 | head -n 1
openssl s_client -connect localhost:8443 -tls1_3
```
> Expected:  
> HTTPS probe succeeds, HTTP endpoint redirects, and TLS 1.3 handshake is established


























