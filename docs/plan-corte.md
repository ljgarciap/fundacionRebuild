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
- [ ] Ronda de QA formal contra los criterios de aceptación de cada proceso
      migrado (no solo verificación técnica de que compila/corre) — con Big
      Bang, esta ronda cubre **todos** los módulos antes del corte, no puede
      quedar ninguno para "después" como sí permitía corte por módulo.
      **Progreso** (empezado 2026-09-05, orden de riesgo: dinero real primero):
      - [x] Ingreso (`IngresoTest.php`)
      - [x] Pago/Pensiones (`PagoTest.php`, `ChargePensionsTest.php`) — **2
            bugs reales encontrados y corregidos**, ver `docs/memory.md`.
      - [x] Tienda/POS (`TiendaTest.php`) — 1 bug real encontrado y corregido.
      - [x] Ahorro (`AhorroTest.php`) — 1 observación abierta (no es bug, ver
            `docs/memory.md`): `salida` no valida contra el acumulado
            disponible, permite dejarlo en negativo. A confirmar con Luis si
            es comportamiento deseado antes del corte.
      - [ ] Agenda, Almuerzo, Auth (login — cobertura de endpoint, más allá
            del mecanismo ya verificado), Bitacora, Concepto, Formatos,
            Minuta, Permiso, Practicante, Reporte, Residente (cambios de
            estado/biometría), Seguimiento, System, Terapia, User — sin
            tocar todavía.
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
