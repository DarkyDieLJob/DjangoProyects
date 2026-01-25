# Receta: Despliegue seguro con healthchecks + migraciones controladas

## Preparación
- Exporta `HOST_PERSIST` y asegúrate de la estructura `persist/` en el host.
- Ajusta `persist/env/.env` (permisos 640, propietario UID/GID del runner).
- Activa perfiles según tu entorno: `db`, `broker` (redis), `worker`.

## Build/Pull
```bash
# Construcción local (no hay registry remoto)
docker compose build
# Si usas imágenes preconstruidas en tu contexto, podrías hacer pull (opcional)
# docker compose pull
```

## Migraciones (previas al up -d)
```bash
docker compose run --rm app python src/manage.py migrate
```

## Arranque (ordenado por healthchecks)
```bash
docker compose up -d [--profile db] [--profile broker] [--profile worker]
```

## Verificación
- `docker compose ps` (espera `healthy` en servicios críticos: db, redis, app, worker si aplica).
- `docker compose logs --no-log-prefix --since=2m` (revisar errores recientes).
- (Si se agrega a futuro) endpoint `/health/` responde 200.

### Perfiles y depends_on opt-in

- Para que `app` espere salud de `db` y `redis`, activar el perfil opt-in `appdeps` además de los perfiles de los servicios requeridos:

```bash
COMPOSE_PROFILES=db,broker,appdeps docker compose up -d
```

Esto utiliza `depends_on: condition: service_healthy` y asegura orden de arranque determinístico.

## Notas
- Healthchecks:
  - db: `pg_isready`
  - redis: `redis-cli ping`
  - app: `python src/manage.py check --deploy` (placeholder minimalista)
  - worker: `python src/manage.py check`
- depends_on con `condition: service_healthy` asegura orden de arranque y tolerancia a latencia.
- Tradeoff: pequeño tiempo extra (migrate + healthchecks) a cambio de mayor estabilidad.

### Inputs del workflow reusable (readiness/smoke)

- `smoke_timeout` (segundos): default 90.
- `smoke_url`: path relativo para smoke (default `/`).
- `smoke_port`: puerto local para smoke HTTP cuando no se define `base_url` (default 8000).
- `base_url`: URL absoluta opcional para smoke; si se define, ignora `smoke_port`.
- `smoke_mode`:
  - `curl_only` (default): smoke HTTP externo sencillo con retries acotados.
  - `in_container_http_then_cmd`: primero espera HTTP dentro del contenedor con retries cortos y luego ejecuta `smoke_check_command` si se provee.
- `cleanup_cache` (bool): default `false`. Si `true`, ejecuta prune de caches Docker al finalizar (más lento; útil para depuración/espacio).

Ejemplos:

```yaml
jobs:
  staging_build_deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      profiles: db,broker,worker
      smoke_mode: curl_only
      smoke_port: 8000
      smoke_url: /

  staging_build_deploy_hijo:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      profiles: db,broker,worker
      smoke_mode: in_container_http_then_cmd
      base_url: http://127.0.0.1:8001
      smoke_url: /
      smoke_check_command: python src/manage.py check
      cleanup_cache: false
```

---

## Recetas de build multi-arch y frontend opcional

### Producción sin frontend (imagen ligera)
```bash
# amd64
docker buildx build --target runtime --platform linux/amd64 -t app:latest .

# ARM
docker buildx build --target runtime --platform linux/arm64 -t app:arm64 .
docker buildx build --target runtime --platform linux/arm/v7 -t app:armv7 .
```

Variables de runtime útiles:
- `NO_FRONTEND=true` (evita pasos de Tailwind en runtime)
- `ENABLE_COLLECTSTATIC=true` (opcional, recolecta estáticos al arrancar)
- `RUN_MAKEMIGRATIONS=false` (si migraciones son 100% externas)

### Staging/producción con frontend (assets precompilados)
```bash
docker buildx build --target runtime-frontend --platform linux/amd64 -t app:with-frontend .
```

Notas:
- El stage `runtime-frontend` solo copia los artefactos generados (p.ej. `static/css/tailwind.css`).
- Node/npm NO están presentes en la imagen final.

### (Opcional) buildx bake (ejemplo conceptual)
```hcl
# docker-bake.hcl (opcional a futuro)
group "default" {
  targets = ["runtime_amd64", "runtime_arm64"]
}

target "runtime_amd64" {
  target = "runtime"
  platforms = ["linux/amd64"]
  tags = ["app:latest"]
}

target "runtime_arm64" {
  target = "runtime"
  platforms = ["linux/arm64"]
  tags = ["app:arm64"]
}
```
Uso:
```bash
docker buildx bake
```

## Rollback simple
1. `docker compose down`
2. Restaurar snapshot de `persist/` (ver guía en docs/INSTALACION.md)
3. `docker compose up -d`

---

## Usando el CLI (atajos)

Ejemplos equivalentes con `project_manage.py`/`Makefile`:

```bash
# Migraciones controladas
python project_manage.py migrate

# Levantar servicios con perfiles
PROFILE=db,broker,worker make up

# Estado y logs
python project_manage.py status
FOLLOW=1 SERVICES=app,worker make logs
```
