# Migrate OnPrem to Cloud
---  

> While running this guide it is optimal to have two terminal tabs: one for docker compose commands and another for Testing each stage.

---

### Stage 01
**Docker Command:** 
```
docker compose --env-file .env -f stages/01-repo/compose.yaml up
```

> Open a new Terminal tab and name it `TEST`. Run all test commands here.  

**Test:** 
- Validate the Database  
```
docker compose --env-file .env -f stages/01-repo/compose.yaml exec -T postgres \
  sh -c 'pg_isready -d "$POSTGRES_DB" -U "$POSTGRES_USER"'
```

