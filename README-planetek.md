# General notes on deploying the stack

# Install Geonode cluster
## Create secret (`geonode-postgres-credentials`)

Decode all keys to find the superuser password:

```bash
kubectl get secret cluster-pg-app \
  -n rheticus-electric-towers \
  -o go-template='{{range $k,$v := .data}}{{$k}}={{$v|base64decode}}{{"\n"}}{{end}}'
```

Create secret:

```bash
kubectl create secret generic geonode-postgres-creds \
  --namespace rheticus-electric-towers \
  --from-literal=postgres_password='<password from above>' \
  --from-literal=geonode_password='geonode-secret' \
  --from-literal=geodata_password='geodata-secret'
```

## Routing

APISIX Routes with TLS

```bash
kubectl create secret tls geonode-tls-secret \
  --namespace rheticus-electric-towers \
  --key="pk_key.key" \
  --cert="pk_pem.pem"
```

Add an Apisix Route.

```yaml
# apisix-route.yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: geonode-route-root
  namespace: rheticus-electric-towers
spec:
  ingressClassName: apisix
  http:
    - name: webgis-root
      match:
        hosts:
          - relect.planetek.it
        paths:
          - "/*"
          - "/"
          - "/**"
      backends:
        - serviceName: webgis
          servicePort: 80
      plugins:
        - name: proxy-rewrite
          enable: true
          config:
            headers:
              X-Forwarded-Proto: "https"
```

Add an ApisixTls resource.

```yaml
# apisix-tls.yaml
apiVersion: apisix.apache.org/v2
kind: ApisixTls
metadata:
  name: geonode-tls
  namespace: rheticus-electric-towers
spec:
  hosts:
    - relect.planetek.it
  ingressClassName: apisix
  secret:
    name: geonode-tls-secret
    namespace: rheticus-electric-towers
```

Apply the resources:

```bash
kubectl apply -f apisix-route.yaml
kubectl apply -f apisix-tls.yaml
```

## Install UI

Apply the resources:

```bash
kubectl apply -f pk-client.yaml
```
```yaml
# pk-client.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webgis
  namespace: rheticus-electric-towers
  labels:
    app: webgis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webgis
  template:
    metadata:
      labels:
        app: webgis
    spec:
      imagePullSecrets:
        - name: rheticus-electric-towers-pull-secret
      tolerations:
        - key: "cluster"
          operator: "Equal"
          value: "rheticuset_cluster"
          effect: "NoSchedule"
      containers:
        - name: webgis
          image: harbor.planetek.it/rheticus_electric_towers/rheticus-electric-towers-web-gis
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3

---
apiVersion: v1
kind: Service
metadata:
  name: webgis
  namespace: rheticus-electric-towers
spec:
  selector:
    app: webgis
  ports:
    - port: 80
      targetPort: 80
      name: http
  type: ClusterIP
```

Create secret.

```bash
kubectl create secret generic webgis-client-creds \
  --from-literal=ADMIN_USERNAME='admin' \
  --from-literal=ADMIN_PASSWORD='geonode' \
  -n rheticus-electric-towers
```

## Install Chart Dependencies

```bash
helm dependency update charts/geonode
```

## Deploy

```bash
# Dry run first
helm upgrade --cleanup-on-fail \
  --install \
  --namespace rheticus-electric-towers \
  --values my-values.yaml \
  --dry-run \
  geonode charts/geonode

# If dry run is clean, apply
helm upgrade --cleanup-on-fail \
  --install \
  --namespace rheticus-electric-towers \
  --values my-values.yaml \
  geonode charts/geonode
```

## Monitor

```bash
# Watch everything
watch kubectl get pods,jobs,pvc -n rheticus-electric-towers

# 1. init-db job — runs Django migrations + sets admin password
kubectl logs -n rheticus-electric-towers \
  job/geonode-geonode-init-db-job -f

# 2. statics job — collects Django static files
kubectl logs -n rheticus-electric-towers \
  job/geonode-geonode-statics-job -f

# 3. Main geonode pod (check both containers: geonode + celery)
kubectl logs -n rheticus-electric-towers \
  pod/geonode-geonode-0 -c geonode -f
```

## Create secret for Dockeregistry

```Bash
kubectl create secret docker-registry rheticus-electric-towers-pull-secret \
  --docker-server=harbor.planetek.it \
  --docker-username=electrictower_usr \
  --docker-password=eleCtr1c_T0weR \
  --namespace rheticus-electric-towers
```




```
export CSRF_TRUSTED_ORIGINS=["https://relect.planetek.it"] \
export ALLOWED_HOSTS=["relect.planetek.it"] \
export CORS_ALLOWED_ORIGINS=["https://relect.planetek.it"] \
export CORS_ALLOW_CREDENTIALS=True \
export CSRF_COOKIE_SECURE=True \
export SESSION_COOKIE_SECURE=True \
export CSRF_COOKIE_SAMESITE=None \
export SESSION_COOKIE_SAMESITE=None \
export USE_X_FORWARDED_HOST = True \
export SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https") 
```

```
kubectl set env statefulset/geonode-geonode \
GEONODE_EXTRA_SETTINGS='ALLOWED_HOSTS=["relect.planetek.it"]
CSRF_TRUSTED_ORIGINS=["https://relect.planetek.it"]
CORS_ALLOWED_ORIGINS=["https://relect.planetek.it"]
CORS_ALLOW_CREDENTIALS=True
CSRF_COOKIE_SECURE=True
SESSION_COOKIE_SECURE=True
CSRF_COOKIE_SAMESITE=None
SESSION_COOKIE_SAMESITE=None
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")' \
-n rheticus-electric-towers
```

```
kubectl rollout restart statefulset geonode-geonode -n rheticus-electric-towers
```

```
kubectl exec geonode-geonode-0 -n rheticus-electric-towers -- /bin/bash -c 'echo -e "$GEONODE_EXTRA_SETTINGS"'
```

# Install Cluster DB

```bash
kubectl apply -f cluster-pg.yaml
```

```yaml
# cluster-pg.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-pg
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgis:15
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  affinity:
     tolerations:
      - key: "cluster"
        operator: "Equal"
        value: "rheticuset_cluster"
        effect: "NoSchedule"
  storage:
    size: 1Gi
```

## Prepare the Databases

```bash
kubectl exec -it cluster-pg-1 \
  -n rheticus-electric-towers \
  -- psql -U postgres
```

```sql
-- Create dedicated users with passwords you control
CREATE USER geonode WITH PASSWORD 'geonode_secret' CREATEDB LOGIN;
CREATE USER geodata WITH PASSWORD 'geodata_secret' CREATEDB LOGIN;

-- Create the two required databases
CREATE DATABASE geonode OWNER geonode;
CREATE DATABASE geodata OWNER geodata;
```

```sql
-- Install PostGIS in geonode db
\c geonode
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO geonode;
GRANT ALL ON SCHEMA public TO geonode;
```

```sql
-- Install PostGIS in geodata db
\c geodata
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO geodata;
GRANT ALL ON SCHEMA public TO geodata;

\q
```

Grant privileges to `app` user.

```sql
GRANT ALL PRIVILEGES ON DATABASE geonode TO app;
GRANT ALL PRIVILEGES ON DATABASE geodata TO app;
\c geonode
GRANT ALL ON SCHEMA public TO app;
\c geodata
GRANT ALL ON SCHEMA public TO app;
\q
```

Make sure the DB matches exactly the given credentials:

```bash
kubectl exec -it cluster-pg-1 \
  -n rheticus-electric-towers \
  -- psql -U postgres -c "ALTER USER geonode WITH PASSWORD 'geonode-secret';"

kubectl exec -it cluster-pg-1 \
  -n rheticus-electric-towers \
  -- psql -U postgres -c "ALTER USER geodata WITH PASSWORD 'geodata-secret';"
```

Verify:

```bash
kubectl exec -it cluster-pg-1 \
  -n rheticus-electric-towers \
  -- psql -U postgres -c "\l"
# You should see geonode and geodata in the list
```
