# Plan de Corte a Producción — Fundación Rebuild

Estado: **decisiones de negocio cerradas, checklist técnica en curso**.
Creado 2026-09-05 al retomar el proyecto — la auditoría de ese día encontró
que no existía ningún plan de corte, staging ni CI/CD, pese a que la
migración de módulos críticos está casi terminada según `consolidado.md`.
Mismo día, daily con Luis: estrategia definida (**Big Bang**, § 2), operación
actual del legado confirmada (§ 1) y plan de rollback acordado (§ 4). Lo que
resta es exclusivamente técnico — ver checklist § 3.

## 1. Operación actual del sistema legado
- [x] **¿Dónde corre hoy en producción?** Confirmado por Luis (daily
      2026-09-05): el mismo hosting de origen del dump (`u727327027_...`,
      cPanel/Hostinger). No hay un hosting distinto que reconciliar.
- [x] **¿Quién administra/tiene acceso? ¿Hay backups automáticos?**
      Confirmado por Luis (daily 2026-09-05): el acceso lo tiene él mismo;
      **no hay backups automáticos** — solo respaldos manuales. Implicación
      para § 4 (rollback): no asumir que existe una copia reciente al momento
      del corte — hay que sacar un dump manual explícito justo antes, como
      paso obligatorio del plan (no un "ya está cubierto").
- [x] **¿Volumen de escritura diaria real?** Confirmado por Luis (daily
      2026-09-05): el legado **sigue operando activo hoy** (ingresos, pagos,
      ventas, seguimientos entrando en vivo) — no está en modo de solo
      consulta. Implicación técnica para § 2: `backend/.env` apunta hoy a
      MySQL **local** (copia del dump, no la base real del hosting) — sea
      cual sea la estrategia elegida, en algún punto el sistema nuevo debe
      apuntar a la base real para no perder lo que el legado sigue generando
      mientras se valida. Esto no elimina ninguna opción de § 2, pero cambia
      su costo relativo: "big bang" evita sostener dos sistemas escribiendo
      sobre la misma base en paralelo; "corte por módulo" y "piloto por sede"
      exigen resolver esa convivencia antes de arrancar, no después.

## 2. Estrategia de corte — **decidido: Big Bang** (Luis, daily 2026-09-05)
Se descartan "corte por módulo" y "piloto por sede" — ambas exigían resolver
la convivencia de los dos sistemas escribiendo en paralelo sobre la base real
(ver nota técnica en § 1), costo que Big Bang evita de raíz.

**Mecánica acordada**: se congela el legado (ventana de downtime, día/horario
a definir), se saca un backup manual explícito de la base real del hosting
(no hay backups automáticos — ver § 1), se apunta `backend/.env` a esa base
real (hoy apunta a MySQL local), se activa el nuevo sistema. Exige que **todos**
los módulos estén validados antes — ver checklist § 3, ahora simplificada al
no necesitar coexistencia entre sistemas.

## 3. Validación previa al corte (bloqueante, sin importar la estrategia elegida)
- [x] Cobertura de tests del proceso de **Ingreso** (multi-tabla, atómico) —
      cerrado 2026-09-05, ver `backend/tests/Feature/IngresoTest.php` (6 tests:
      creación atómica cross-tabla, reenvío sin duplicar cargos, acudiente
      existente conserva su rol de staff, rollback completo si falta
      `guardian_data`). Biometría (firma/huella) queda fuera de este archivo —
      no tiene test dedicado todavía, ver nota abajo.
- [x] Resolver `actores` y `efectivo` (ver `consolidado.md` — dos tablas
      "Pendiente" sin decisión de si siguen vigentes en la operación real).
- [x] Confirmar mecanismo real de migración de contraseñas MD5 → Bcrypt —
      cerrado 2026-09-05: `AuthController::login` intenta Bcrypt primero y cae
      a MD5 contra `validacion.password` (ahí vivía el hash legacy real, no en
      `usuarios.password`), con upgrade silencioso a Bcrypt al validar por esa
      vía. Transparente para el usuario, cumple la regla del `CLAUDE.md`. Doc
      de `validacion` corregida en `consolidado.md` (describía solo su función
      biométrica, no la de password legacy).
- [ ] Cobertura de tests de **biometría en Ingreso** (firma/huella, tabla
      `validacion`) — no incluida en `IngresoTest.php`, sigue pendiente.
- [ ] Ambiente de staging con **una copia fresca de la base real del hosting**
      (no el dump original de referencia, que ya tiene 8 años — el legado
      sigue escribiendo hoy, ver § 1) antes del corte. Con Big Bang decidido,
      este paso es más crítico que antes: no hay módulos escalonados que
      absorban sorpresas, así que la última validación tiene que verse contra
      datos reales y actuales, no contra el dump histórico.
- [x] **Ronda de QA formal — completa, 24/24 controllers** (empezado y
      cerrado el 2026-09-05, orden de riesgo: dinero real primero). 147
      tests nuevos en total, 153/153 pasan en la suite completa.
      **6 bugs reales encontrados y corregidos** (todos crasheaban en MySQL
      real, enmascarados por SQLite en los tests hasta escribirlos contra el
      schema correcto — ver detalle completo en `docs/memory.md`):
      1. `pensions:charge` (cron diario) — relación `residente()` faltante
         en `CobroPension`.
      2. `TiendaController::storeSale` — sin validación de stock.
      3. `ResidenteController::history()` — columna inexistente en
         `abonopensiones`.
      4. `agenda.encargado` — columna nunca migrada.
      5. `AgendaController::index()` — relación `residente()` faltante en
         `Agenda`.
      6. `ReporteController::inventory()` — columnas inexistentes en
         `productos` (mismo patrón que el bug de Tienda).
      Módulos sin bugs (solo cobertura agregada): Ingreso, Ahorro,
      Almuerzo, Auth, Minuta, Permiso, Seguimiento, Terapia, Practicante,
      Concepto, Bitacora, User, System, Formatos, Reporte.
      **Excepciones documentadas, no bugs**: `SystemController::getStatus()`
      e `incomeByMonth()`/parte de `seguimientosPsicologia()` usan sintaxis
      SQL específica de MySQL (`SUBSTRING_INDEX`, `DATE_FORMAT`) que SQLite
      no soporta — sin test unitario por incompatibilidad de motor, pendiente
      de validar a mano contra MySQL real en staging (ver ítem siguiente).
      **Observación cerrada** (Ahorro, no era bug): `salida` no valida contra
      el acumulado disponible — decisión de Luis (2026-09-06): se deja así,
      comportamiento intencional.
      **Pendiente fuera de esta ronda**: biometría en Ingreso (firma/huella)
      no tiene test dedicado.
- [x] **Validación manual Angular ↔ backend local, contra base de datos real**
      (2026-09-06) — ver `docs/memory.md` para el detalle completo del
      ambiente (Docker MySQL dedicado en :3307, dump real de 742 residentes/
      777 usuarios importado, backend en :8010, frontend en :4200, validado
      con Playwright CLI). **1 bug real de severidad alta encontrado y
      corregido**: `Api.handleApiError()` (frontend) redirigía la página
      completa ante cualquier 401, incluido el del propio login fallido —
      un usuario real nunca veía "Credenciales inválidas", solo la pantalla
      recargándose en blanco. **Hallazgo más amplio, sin corregir, para
      auditoría aparte**: 17 de 24 componentes que usan `.subscribe()` no
      inyectan `ChangeDetectorRef` pese a que es patrón obligatorio bajo
      `provideZonelessChangeDetection()` (ya documentado en `memory.md`,
      sesión 18-May) — riesgo de estado que no se refleja en pantalla tras
      una respuesta async, en cualquiera de esos 17.
- [x] **Auditoría de `ChangeDetectorRef` faltante — completa** (2026-09-06,
      ver `docs/memory.md`). De los 24 componentes que usan `.subscribe()`,
      los 16 que no inyectaban `ChangeDetectorRef` ahora lo hacen (login.ts
      ya se había corregido antes). **2 bugs reales confirmados en vivo con
      Playwright** durante el audit — mismo patrón que el bug de login:
      `usuarios.ts` (error de "documento ya existe" al crear/editar un
      usuario nunca se mostraba al administrador) y `conceptos.ts` (error
      409 de "concepto en uso, no se puede eliminar" tampoco se mostraba).
      El resto de los 16 recibió el mismo fix preventivo por patrón de
      código, sin confirmación individual con Playwright de cada uno.
      Build de producción de Angular limpio tras el fix.
- [x] **Comparación en vivo legado vs. sistema nuevo — Ingreso y Pensiones**
      (2026-09-06, pedido por Luis: "¿el sistema nuevo cubre todo lo del
      legado?", empezando por los 2 módulos con más reclamos reales). Ver
      `docs/memory.md` para el detalle completo.
      - **Ingreso**: **bug crítico encontrado y corregido** —
        `POST /api/ingresos` crasheaba SIEMPRE por una columna
        (`tipo_sanguineo`) nunca migrada. El proceso más crítico del
        sistema completo estuvo roto de punta a punta sin que ningún test
        lo detectara. Verificado end-to-end tras el fix: 200 OK, residente
        real creado con las 8 tablas relacionadas.
      - **Pensiones**: sin bugs — paridad exacta confirmada contra 10
        residentes reales (mismo cálculo de saldo, mismos valores,
        incluidos casos de saldo negativo) y un abono real registrado por
        la UI nueva y releído desde el legado con el mismo resultado.
      - **Pendiente**: repetir este mismo ejercicio (legado vivo + sistema
        nuevo vivo + Playwright) para el resto de los módulos críticos —
        es la única metodología que demostró atrapar bugs reales de este
        tipo (los backend tests solos no bastan, ver caso de Ingreso).
- [ ] Backup manual explícito de la base real, tomado justo antes de la
      ventana de corte (ver § 1 — no hay backups automáticos).
- [ ] Fecha/horario de la ventana de corte (a definir con Luis).

## 4. Plan de rollback — **decidido** (Luis, daily 2026-09-05)
- [x] **Ventana de solo-lectura**: el legado se mantiene disponible en modo
      solo-lectura **1 semana** después del corte, por si aparece un caso no
      contemplado.
- [x] **Decisión de rollback**: la toma **Luis**, sin un criterio formal
      pre-establecido — a su juicio en el momento, caso por caso.

---
Las 3 secciones de decisión de negocio (§ 1, § 2, § 4) quedaron **cerradas**
en el daily del 2026-09-05. Lo único pendiente en este documento es la
checklist técnica de § 3 (staging con copia fresca de la base real, QA formal
completa, backup manual pre-corte, fecha/horario de la ventana — este último
también a definir con Luis más adelante, cuando § 3 esté lista).

**Próximo paso**: cerrar la checklist técnica de § 3 (staging, QA formal,
backup manual, fecha de ventana) — sin más decisiones de negocio bloqueantes
de por medio.
categoría "decisión de negocio").
