# Informe de auditoría de dependencias (seguridad)

**Proyecto:** Nutribiotics  
**Fecha:** 2026-03-25  
**Alcance:** solo dependencias npm (ayuda del monorepo en la raíz, `frontend/`, `backend/`). No se revisaron controles a nivel de aplicación (CORS, autenticación, límites de peticiones, secretos).

## Entorno

| Herramienta | Versión (usada en esta auditoría) |
| ----------- | --------------------------------- |
| Node        | v22.18.0                          |
| npm         | 10.x (incluido con Node 22)       |

## Metodología

1. **Lockfiles:** se aseguró que cada espacio de trabajo tenga un `package-lock.json` versionado. El backend no tenía uno; se generó con `npm install` y se comprobó con `npm ci`.
2. **Línea base:** se ejecutó `npm ci` (donde existía lockfile) o `npm install`, y después `npm audit`.
3. **Remediación automática:** se ejecutó `npm audit fix` en `frontend/` y `backend/` (solo actualizaciones compatibles con semver).
4. **Overrides puntuales:** para hallazgos sin arreglo automático seguro, se aplicaron `overrides` a nivel raíz del `package.json` correspondiente (ver abajo), luego `npm install`, `npm audit` y builds de producción.
5. **Verificación:** `npm run build` en `frontend/` y `backend/` tras los cambios.

## Resumen

| Espacio de trabajo | Vulnerabilidades antes | Solo tras `npm audit fix` | Final (tras overrides) |
| ------------------ | ---------------------- | ------------------------- | ------------------------ |
| Raíz               | 0                      | 0                         | 0                        |
| Frontend           | 10                     | 2 moderadas               | 0                        |
| Backend            | 25                     | 8 moderadas               | 0                        |

Los totales del backend en la primera columna corresponden al primer resumen de `npm audit` justo después de crear `backend/package-lock.json` (antes de cualquier arreglo): **1 crítica, 10 altas, 12 moderadas, 2 bajas**.

## Frontend — hallazgos relevantes (línea base)

Todos eran transitivos salvo indicación contraria. Las fuentes son URLs de GitHub Security Advisory de `npm audit` en la línea base.

| Paquete / cadena | Severidad | Aviso / tema |
| ---------------- | --------- | ------------ |
| `@remix-run/router` | alta | [GHSA-2w69-qvjg-hvjx](https://github.com/advisories/GHSA-2w69-qvjg-hvjx) — React Router / relacionado con redirección abierta |
| `react-router`, `react-router-dom` | alta | Mismo problema aguas arriba vía el router |
| `rollup` | alta | [GHSA-mw96-cpmx-2vgc](https://github.com/advisories/GHSA-mw96-cpmx-2vgc) — path traversal / escritura arbitraria de archivos |
| `minimatch` | alta | [GHSA-3ppc-4f35-3m26](https://github.com/advisories/GHSA-3ppc-4f35-3m26) — ReDoS |
| `flatted` | alta | [GHSA-25h7-pfq9-p65f](https://github.com/advisories/GHSA-25h7-pfq9-p65f) — DoS por recursión descontrolada en `parse()` |
| `esbuild` (vía Vite) | moderada | [GHSA-67mh-4wv8-2f99](https://github.com/advisories/GHSA-67mh-4wv8-2f99) — manejo de peticiones en el servidor de desarrollo |
| `vite` | moderada | Arrastraba un rango vulnerable de `esbuild` |
| `ajv` | moderada | [GHSA-2g4f-4pwh-qvx6](https://github.com/advisories/GHSA-2g4f-4pwh-qvx6) — ReDoS con `$data` |
| `lodash` | moderada | [GHSA-xxjr-mmjv-4gpg](https://github.com/advisories/GHSA-xxjr-mmjv-4gpg) — contaminación de prototipo (`_.unset` / `_.omit`) |

Tras `npm audit fix`, quedaron **dos** hallazgos **moderados**: `esbuild` anidado bajo `vite@5.4.x` (Vite 5 fija un `esbuild` más antiguo; `npm audit fix --force` habría saltado a Vite 8, lo cual se evitó).

## Backend — hallazgos relevantes

**Tras el primer `npm audit fix` (antes de overrides),** quedaron ocho hallazgos **moderados**:

| Tema | Severidad | Aviso | Cadena de dependencias |
| ---- | --------- | ----- | ---------------------- |
| `ajv` ReDoS | moderada | [GHSA-2g4f-4pwh-qvx6](https://github.com/advisories/GHSA-2g4f-4pwh-qvx6) | `@nestjs/cli` / `@nestjs/schematics` → `@angular-devkit/*` (herramientas de desarrollo) |
| `langsmith` SSRF (cabecera de tracing) | moderada | [GHSA-v34v-rq6j-cj6p](https://github.com/advisories/GHSA-v34v-rq6j-cj6p) | `@langchain/core` (transitivo, p. ej. vía `@browserbasehq/stagehand`) |

Los hallazgos **críticos** y **altos** iniciales del backend se resolvieron con actualizaciones compatibles del lockfile durante `npm audit fix` (sin `npm audit fix --force`).

## Correcciones aplicadas

| Acción | Ubicación | Detalles |
| ------ | --------- | -------- |
| Nuevo lockfile | `backend/package-lock.json` | Generado con `npm install`; instalaciones reproducibles y auditorías fiables. |
| `npm audit fix` | `frontend/`, `backend/` | Dependencias anidadas actualizadas dentro de los rangos semver existentes (p. ej. `react-router-dom`, `rollup`, `vite` parche/menor). |
| `overrides` | [frontend/package.json](frontend/package.json) | `"esbuild": "^0.25.12"` — fuerza un `esbuild` parcheado para todos los consumidores (incluido bajo Vite 5) sin subir a Vite 8. |
| `overrides` | [backend/package.json](backend/package.json) | `"ajv": "^8.18.0"` — corrige `ajv` vulnerable anidado en el árbol Nest/Angular devkit. `"langsmith": "^0.5.13"` — rango parcheado del SDK LangSmith manteniendo `@langchain/core` 0.3.x. |
| `engines` | [backend/package.json](backend/package.json) | `"node": ">=20.0.0"` — alineado con el frontend y Node LTS soportado. |

**Verificación de build:** `npm run build` correcto en `frontend/` y `backend/` con los lockfiles y overrides finales. **Verificación de instalación:** `npm ci` correcto en `frontend/` y `backend/`.

## Riesgo residual / aceptado

- **`overrides` de npm:** fijar versiones transitivas puede diferir de lo que declaran formalmente los paquetes originales. Si una versión futura de Vite 5, Nest CLI o LangChain endurece rangos incompatibles, revisar los overrides y priorizar actualizaciones oficiales.
- **Avisos de dependencias peer:** `npm install` en el backend puede seguir advirtiendo sobre peers opcionales de `zod` bajo el `openai` anidado de `@browserbasehq/stagehand` (el proyecto usa `zod@4` en la raíz). Es el mismo comportamiento funcional que antes de la auditoría; no es un hallazgo de `npm audit`.
- **Paquetes obsoletos (deprecated):** npm puede marcar paquetes obsoletos (p. ej. `glob` antiguo en dependencias transitivas). La obsolescencia no siempre implica un problema de seguridad; llevarla aparte de `npm audit`.

## Recomendaciones aplicadas

Los siguientes puntos de la lista de recomendaciones ya están implementados en el repositorio:

| # | Recomendación | Implementación |
| --- | ------------- | -------------- |
| 1 | CI: `npm ci` + `npm audit --audit-level=high` por espacio de trabajo | [`.github/workflows/npm-security-audit.yml`](.github/workflows/npm-security-audit.yml) en `push` / `pull_request` a `main` o `master` para raíz, `frontend/` y `backend/`. Los jobs de frontend y backend también ejecutan `npm run build` tras una auditoría limpia. Usa Node **20** (LTS) y `actions/checkout@v4` con **`submodules: recursive`** para que un submódulo `frontend` esté presente en CI. |
| 2 | Dependabot: actualizaciones npm agrupadas | [`.github/dependabot.yml`](.github/dependabot.yml) — actualizaciones npm semanales con un PR agrupado por espacio de trabajo (`/`, `/frontend`, `/backend`). |
| 4 | `engines.node` en la raíz | [package.json](package.json) — `"engines": { "node": ">=20.0.0" }` alineado con `frontend/` y `backend/`. |

**Operativo (no automatizado en el repo):** volver a auditar tras actualizaciones mayores (`vite`, `@nestjs/cli`, `@browserbasehq/stagehand`, etc.) y endurecer CI con `--audit-level=moderate` si se desea un umbral más estricto que «alta y superior».

## Reauditoría (tras aplicar recomendaciones)

**Fecha:** 2026-03-25 (seguimiento)

| Espacio de trabajo | Comando | Resultado |
| ------------------ | ------- | --------- |
| Raíz | `npm ci && npm audit --audit-level=high` | 0 vulnerabilidades (código de salida 0) |
| Frontend | `npm ci && npm audit --audit-level=high` | 0 vulnerabilidades (código de salida 0) |
| Backend | `npm ci && npm audit --audit-level=high` | 0 vulnerabilidades (código de salida 0) |

Herramientas locales en esa ejecución: **Node v22.18.0**, **npm 11.6.0**. GitHub Actions usa **Node 20** según el workflow.

## Recomendaciones (lista histórica)

1. ~~**CI:** Ejecutar `npm ci && npm audit --audit-level=high`~~ — **Hecho** (ver [Recomendaciones aplicadas](#recomendaciones-aplicadas)).
2. ~~**Dependabot / Renovate:** actualizaciones npm agrupadas~~ — **Hecho** ([`.github/dependabot.yml`](.github/dependabot.yml)).
3. **Cadencia de reauditoría:** repetir este proceso tras actualizaciones mayores de dependencias (especialmente `vite`, `@nestjs/cli`, `@browserbasehq/stagehand`).
4. ~~**`engines` en la raíz:**~~ — **Hecho** ([package.json](package.json)).

## Fuera de alcance (este repaso)

- Endurecimiento en tiempo de ejecución e infraestructura (política CORS, almacenamiento de JWT, exposición Redis/Mongo, cabeceras, limitación de tasa).
- Imágenes de contenedor, paquetes del SO o cadena de suministro no npm.

---

*Regenerar o actualizar este documento cuando el árbol de dependencias cambie de forma relevante.*
