# Release y uso de gh (resumen para agentes)

## Requisitos
- gh instalado y autenticado: `gh auth login`.
- Estar en la rama correcta:
  - PR: `pre-release` -> `main`.
  - Release: `main` (ya actualizado con el PR).

## Crear PR (pre-release -> main)
- Comando simple (evita problemas de quoting):
  ```bash
  gh pr create --base main --head pre-release \
    --title "fix(gitignore): ignore local tmp_* deployment folders" \
    --body "Ignora carpetas temporales tmp_* (tmp_host/tmp_persist)."
  ```
- Para cuerpos extensos, preferir `--body-file`:
  ```bash
  BODY_FILE=$(mktemp)
  cat > "$BODY_FILE" <<PRBODY
  ...cuerpo del PR...
  PRBODY
  gh pr create --base main --head pre-release --title "..." --body-file "$BODY_FILE"
  ```

## Pitfalls observados
- Quoting en bash con here-docs y backticks puede romper comandos multilinea.
  - Solución: usar `--body-file` o mensajes simples.
- Evitar subshells complejos en una sola línea; dividir pasos.
- Verificar rama actual con `git rev-parse --abbrev-ref HEAD` antes de `gh`.

## Release en main
- Este repo dispara el workflow de GitHub Releases al pushear tags `v*.*.*`.
- Si no hay configuración de `standard-version` en la raíz, se puede crear un tag patch manualmente:
  ```bash
  git tag -a vX.Y.Z -m "release: vX.Y.Z"
  git push origin vX.Y.Z
  ```
- Recomendado: integrar `standard-version` para generar `CHANGELOG.md` y tags automáticamente.

## Flujo posterior
- Sincronizar ramas tras release si aplica (ver docs/GIT_AGENTES.md).
