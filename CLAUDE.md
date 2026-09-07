# Fundación Rebuild

Refactoring del sistema monolítico PHP/MySQL de la fundación (residencial de
rehabilitación, sedes JOREC y Jesús es mi Roca) a una arquitectura moderna,
**sin alterar el esquema de base de datos legado** (8 años de antigüedad,
50 tablas, dump de referencia en `fundacion/u727327027_fjemr.sql`).

## Stack
- **Backend**: Laravel (`backend/`, repo propio — remoto `fundacionBackend`).
- **Frontend**: Angular (`frontend/`, repo propio — remoto `fundacionFrontend`).
- **Base de datos**: MySQL/MariaDB, esquema original preservado (no se migra
  a un esquema nuevo — Laravel lee/escribe directo sobre las tablas legadas).
- **Legado**: PHP monolítico original en `fundacion/` (repo propio — remoto
  `fundacionOld`, subcarpeta `fundacion/ingreso/`; corregido 2026-09-05, el
  remoto local apuntaba mal a `fundacionMonolithic`). Se conserva como
  referencia funcional mientras dure la migración — no se modifica.

**Nota de versiones**: verificar versión real instalada antes de asumir la de
la doc — `backend/composer.json` y `frontend/package.json` son la fuente de
verdad, no lo que diga este archivo ni `docs/`.

## Estructura de repos (deliberada, no consolidar sin decisión explícita)
Este repo raíz versiona **solo `docs/` y metadata del proyecto** — `backend/`
y `frontend/` son repos git independientes con remoto propio (ver `.gitignore`).
Se evaluó consolidar a monorepo (patrón usado en CardGame/Proseguir) y se
decidió mantener los 3 repos separados (2026-09-05) — no revisar esta decisión
sin planteárselo a Luis explícitamente, implica migrar historial y remotos.

## Estado y memoria del proyecto
- `docs/architecture.md` — diseño técnico, stack, roles y estrategia de seguridad.
- `docs/consolidado.md` — mapa de cobertura de las 50 tablas legacy vs. lo
  migrado (fuente de verdad de qué está "Consolidado" / "Pendiente" / "Soporte").
- `docs/memory.md` — bitácora de sesiones de trabajo previas.
- `docs/plan-corte.md` — plan de corte a producción (esqueleto, con preguntas
  de negocio abiertas — ver sección "Pendiente de Luis" adentro).

**Auditoría de retoma (2026-09-05)**: se verificó el estado real contra el
código (no solo contra lo que decían los `.md`) — repos backend/frontend
limpios y con remoto, 15 tests backend pasan, build de Angular en producción
compila sin errores. Detalle completo en el historial de esa sesión, ver
`docs/memory.md`.

**Estado real de cobertura (actualizado 2026-09-06, no confiar en "50/50"
sin mirar `docs/consolidado.md`)**: tras una ronda de QA formal (147 tests
nuevos) y una comparación en vivo legado-vs-nuevo módulo por módulo, se
encontraron y corrigieron **8 bugs reales** (backend, frontend y de
comportamiento de negocio — ver `docs/memory.md` para el detalle completo
de cada uno) y se confirmó que **Tienda POS y Compras/Proveedores corren
en la práctica sobre una base de datos distinta y separada**
(`u727327027_tienda`, con actividad hasta hoy) — la migrada
(`u727327027_fjemr`) para esos 2 módulos está desconectada de la operación
real desde antes de 2020. Esos 2 módulos quedaron re-marcados "Pendiente
(real)" en `docs/consolidado.md` (antes "Consolidado"/"Migrado"), pausados
por decisión de Luis hasta encararlos como su propio ciclo de trabajo. El
mecanismo MD5→Bcrypt (línea de abajo) también quedó confirmado y correcto
en esa misma ronda.

## Reglas específicas de este proyecto (además de las globales del workspace)
- **Nunca alterar el esquema de las tablas legadas** (agregar columnas está
  permitido si es necesario para el nuevo sistema — ej. campos de biometría en
  `residentes` — pero nunca renombrar/eliminar columnas ni tablas existentes
  sin decisión explícita de Luis, para no romper compatibilidad con el legado).
- Migración de contraseñas MD5 (legado) → Bcrypt (Laravel) debe ser
  transparente para el usuario final — mecanismo verificado y confirmado
  correcto en `AuthController::login` (2026-09-05): intenta Bcrypt primero,
  cae a MD5 contra `validacion.password` (ahí vive el hash legacy real) con
  upgrade silencioso a Bcrypt.
- Todo cambio que toque una tabla marcada "Pendiente" en `docs/consolidado.md`
  requiere primero decisión de negocio de Luis sobre si ese proceso sigue
  vigente en la operación real de la fundación (ver `actores`, `efectivo`,
  y — desde 2026-09-06 — Tienda POS/Compras, que corren sobre una base
  separada nunca migrada).
- `.env` de `backend/` apunta hoy a MySQL local — no hay ambiente de
  staging/producción configurado todavía (ver `docs/plan-corte.md`).
