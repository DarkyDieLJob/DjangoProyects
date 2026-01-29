# Guía de adopción ADR0025 (Imágenes inmutables y despliegue vía GHCR)

Esta guía resume cómo aplicar el ADR0025 en proyectos hijos usando los workflows reusables del repositorio padre.

## 1) Parametrizar docker compose

- Reemplaza `build:` por `image:` en los servicios `app` y `worker`:
  - `image: ${IMAGE_REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}`
- Variables requeridas:
  - `IMAGE_REGISTRY=ghcr.io`
  - `IMAGE_REPO=<org>/<app>` (ej.: `DarkyDieLJob/gestionferreteria`)
  - `IMAGE_TAG=<staging-latest | test-YYYYMMDD-N | vX.Y.Z>`

## 2) Publicar imágenes (Build & Push)

Llama al reusable del padre `.github/workflows/reusable-build-push.yml` desde el hijo, por ejemplo en `pre-release` y tags de release:

```yaml
name: Build & Push
on:
  push:
    branches: [ pre-release ]
  workflow_dispatch:

jobs:
  build_push:
    uses: DarkyDieLJob/DjangoProyects/.github/workflows/reusable-build-push.yml@pre-release
    with:
      image_repo: DarkyDieLJob/gestionferreteria
      tags: staging-latest,sha-${{ github.sha }}
      context: .
      dockerfile: Dockerfile
      platforms: linux/amd64,linux/arm64
    secrets:
      GHCR_PAT: ${{ secrets.GHCR_PAT }}
```

Para release/tag:

```yaml
with:
  image_repo: DarkyDieLJob/gestionferreteria
  tags: vX.Y.Z
```

## 3) Desplegar (Pull + Up, sin build)

Usa el reusable `.github/workflows/reusable-pull-deploy.yml` del padre, parametrizando el tag:

```yaml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: Tag a desplegar
        required: true
        type: string

jobs:
  deploy:
    uses: DarkyDieLJob/DjangoProyects/.github/workflows/reusable-pull-deploy.yml@pre-release
    with:
      runner_label: self-hosted
      image_registry: ghcr.io
      image_repo: DarkyDieLJob/gestionferreteria
      image_tag: ${{ inputs.image_tag }}
      compose_files: docker-compose.yml
      deny_latest: true
    secrets:
      GHCR_PAT: ${{ secrets.GHCR_PAT }}
```

- Staging: `image_tag=staging-latest` o un `test-YYYYMMDD-N` específico.
- Producción: `image_tag=vX.Y.Z` (no usar `latest`).

## 4) Secretos requeridos

- `GHCR_PAT` con permisos:
  - Build & Push: `write:packages`
  - Deploy: `read:packages`

## 5) Validación

- Healthcheck de servicios en compose y smoke test HTTP integrado en el reusable.
- Métricas recomendadas:
  - Tiempo de deploy (< 2 min desde disparo hasta healthy)
  - Tasa de fallos en deploy por build (idealmente 0)

## Notas de compatibilidad

- El flujo de Deploy in-situ previo queda para compatibilidad. Para ADR0025 usa SIEMPRE el Deploy por pull con imagen publicada.
- En producción, habilita `deny_latest: true` (default) para evitar despliegues accidentales con `latest`.
