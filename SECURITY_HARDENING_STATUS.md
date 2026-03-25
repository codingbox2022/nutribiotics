# Endurecimiento de seguridad — estado actual

**A fecha de:** 2026-03-25 (UTC)  
**Alcance:** higiene de dependencias npm, instalaciones reproducibles y comprobaciones automatizadas. **No** cubre la seguridad de la aplicación (CORS, autenticación, secretos, infraestructura).

---

## Postura ante vulnerabilidades (`npm audit`)

Medido frente a los **lockfiles confirmados en el repositorio** (ejecutar localmente `npm audit`; los resultados coinciden con el umbral de `npm audit --audit-level=high`).

| Espacio de trabajo | Críticas | Altas | Moderadas | Bajas | Total |
| ------------------ | -------: | ----: | --------: | ----: | ----: |
| Raíz (`/`) | 0 | 0 | 0 | 0 | **0** |
| Frontend (`/frontend`) | 0 | 0 | 0 | 0 | **0** |
| Backend (`/backend`) | 0 | 0 | 0 | 0 | **0** |

**Herramientas usadas para esta instantánea:** Node v22.18.0, npm 11.6.0.

Para actualizar: en cada directorio, ejecutar `npm ci && npm audit`.

---

## Qué está implementado

| Área | Estado actual |
| ---- | ------------- |
| **Lockfiles** | `package-lock.json` (raíz), `frontend/package-lock.json`, `backend/package-lock.json` — las versiones quedan fijadas para CI y auditorías. |
| **Versión de Node** | Raíz, frontend y backend declaran `"engines": { "node": ">=20.0.0" }`. |
| **Correcciones transitivas** | **Frontend:** en `package.json`, `overrides` → `esbuild` `^0.25.12` (herramientas de build parcheadas bajo Vite 5). **Backend:** `overrides` → `ajv` `^8.18.0`, `langsmith` `^0.5.13`. |
| **CI** | [`.github/workflows/npm-security-audit.yml`](.github/workflows/npm-security-audit.yml): en `push` / `pull request` hacia `main` o `master`, ejecuta `npm ci`, `npm audit --audit-level=high` y `npm run build` en frontend y backend. El checkout usa `submodules: recursive` si el frontend es un submódulo. **Node 20** en Actions. |
| **Dependabot** | [`.github/dependabot.yml`](.github/dependabot.yml): actualizaciones npm semanales agrupadas para `/`, `/frontend` y `/backend`. |

---

## Expectativas continuas

- Volver a ejecutar auditorías (o confiar en CI) tras actualizaciones **mayores** de dependencias.
- Si `frontend` solo existe como **submódulo (gitlink)** en GitHub, Dependabot puede no ver `frontend/package.json` en el repositorio padre; usar Dependabot en el repositorio del submódulo o mantener los manifiestos en el repositorio principal.
- Los `overrides` de npm pueden apartarse de los rangos declarados por los paquetes originales; revisarlos al actualizar Vite, Nest CLI o el ecosistema LangChain.

---

## Documento relacionado

Los hallazgos históricos, referencias a avisos y pasos de remediación del primer repaso están resumidos en [SECURITY_DEPENDENCY_REPORT.md](SECURITY_DEPENDENCY_REPORT.md).
