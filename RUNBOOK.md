# Runbook

## Variables

```bash
export IMAGE=<image repository>
export VERSION=<candidate tag>
export PREVIOUS=<previous tag>
export CONTAINER=hiking-app
export VOLUME=hiking-data
export DJANGO_SECRET_KEY=$(openssl rand -hex 32) # Reuse for upgrade/rollback
export API_TOKEN=$(openssl rand -hex 32) # Reuse for upgrade/rollback
export DATABASE_PATH=/data/db.sqlite3
export PORT=8000
```

## Configuration

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `DJANGO_SECRET_KEY` | yes | – | Django secret key for cryptographic signing |
| `API_TOKEN` | yes **(new)** | – | Shared secret for write access to the API (X-API-Token header) |
| `DATABASE_PATH` | no | /data/db.sqlite3 | Path to the SQLite database file |
| `PORT` | no | 8000 | Port the application listens on |
| `DJANGO_DEBUG` | no | 0 | Enable Django debug mode |
| `DJANGO_ALLOWED_HOSTS` | no | * | Comma-separated list of allowed hosts |
| `GUNICORN_WORKERS` | no | 2 | Number of Gunicorn worker processes |

## Backup

**B1. Backup the database**

```bash
docker cp $CONTAINER:/data/db.sqlite3 db.sqlite3.bak
```

Expected: db.sqlite3.bak created

## Deploy

**D1. Fresh install of the candidate version**

```bash
docker run -d --name $CONTAINER -p $PORT:$PORT -v $VOLUME:/data -e DJANGO_SECRET_KEY=$DJANGO_SECRET_KEY -e API_TOKEN=$API_TOKEN -e DATABASE_PATH=$DATABASE_PATH -e PORT=$PORT $IMAGE:$VERSION
```

Expected: container id

## Upgrade

**U1. Upgrade from previous version**

```bash
docker stop $CONTAINER
docker rm $CONTAINER
docker run -d --name $CONTAINER -p $PORT:$PORT -v $VOLUME:/data -e DJANGO_SECRET_KEY=$DJANGO_SECRET_KEY -e API_TOKEN=$API_TOKEN -e DATABASE_PATH=$DATABASE_PATH -e PORT=$PORT $IMAGE:$VERSION
```

Expected: container id

## Migration

**M1. Verify database migrations were applied**

```bash
curl -fsS http://localhost:$PORT/health/ | grep '\"pending_migrations\": 0'
```

Expected: "pending_migrations": 0

## Health Check

**H1. Verify service health and version**

```bash
curl -fsS http://localhost:$PORT/health/ | grep '"status": "ok"'
curl -fsS http://localhost:$PORT/health/ | grep '"version": "'${VERSION#v}'"'
```

Expected: "status": "ok"

## Rollback

**R1. Rollback to the previous version**

```bash
docker stop $CONTAINER
docker rm $CONTAINER
docker run --rm -v $VOLUME:/data -e DJANGO_SECRET_KEY=$DJANGO_SECRET_KEY -e API_TOKEN=$API_TOKEN $IMAGE:$VERSION python manage.py migrate routes 0002
docker run -d --name $CONTAINER -p $PORT:$PORT -v $VOLUME:/data -e DJANGO_SECRET_KEY=$DJANGO_SECRET_KEY -e DATABASE_PATH=$DATABASE_PATH -e PORT=$PORT $IMAGE:$PREVIOUS
```

Expected: container id
