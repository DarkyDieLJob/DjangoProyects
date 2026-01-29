# DjangoProyects (plantilla)

Plantilla base para proyectos Django con arquitectura limpia, CLI unificado y dockerización lista para producción.

- Documentación completa: ver [docs/DJANGOPROYECTS.md](docs/DJANGOPROYECTS.md)
- Instalación y comandos rápidos: [docs/INSTALACION.md](docs/INSTALACION.md)

Notas importantes:
- Persistencia: usamos `persist/` por host (ver sección correspondiente en docs).
- Este README del padre es genérico. Los hijos deben mantener su propio README específico y NO se sincroniza desde el padre (evita ruido en diffs). 

## Agentes y gh
- Ver `docs/RELEASE_WITH_GH.md` para flujo de PRs con gh y release por tags.
 - Adopción ADR0025 (imágenes inmutables + GHCR): guía en [docs/ADR0025_ADOPCION.md](docs/ADR0025_ADOPCION.md)
   - Workflows reusables provistos por el padre:
     - `.github/workflows/reusable-build-push.yml`
     - `.github/workflows/reusable-pull-deploy.yml`
