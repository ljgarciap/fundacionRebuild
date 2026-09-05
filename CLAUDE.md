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
compila sin errores. Gaps encontrados: cobertura de tests desbalanceada (nada
sobre el proceso de Ingreso, el más crítico), tablas `actores` y `efectivo`
sin resolver, y ausencia total de plan de corte/staging/CI-CD hasta esta
fecha. Detalle completo en el historial de la sesión de esa fecha.

## Reglas específicas de este proyecto (además de las globales del workspace)
- **Nunca alterar el esquema de las tablas legadas** (agregar columnas está
  permitido si es necesario para el nuevo sistema — ej. campos de biometría en
  `residentes` — pero nunca renombrar/eliminar columnas ni tablas existentes
  sin decisión explícita de Luis, para no romper compatibilidad con el legado).
- Migración de contraseñas MD5 (legado) → Bcrypt (Laravel) debe ser
  transparente para el usuario final — verificar el mecanismo real en
  `AuthController` antes de tocar login (no confirmado en la auditoría del
  2026-09-05, queda como pendiente de revisión de Cybersecurity).
- Todo cambio que toque una tabla marcada "Pendiente" en `docs/consolidado.md`
  requiere primero decisión de negocio de Luis sobre si ese proceso sigue
  vigente en la operación real de la fundación (ver `actores`, `efectivo`).
- `.env` de `backend/` apunta hoy a MySQL local — no hay ambiente de
  staging/producción configurado todavía (ver `docs/plan-corte.md`).
