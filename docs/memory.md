# Memoria del Proyecto - Fundación Rebuild

## Estado Actual
Migración completada de los módulos críticos del sistema PHP legado a una arquitectura moderna (Laravel 11 + Angular 19). El sistema es ahora 100% funcional para la operación diaria, con alta fidelidad visual y seguridad mejorada.

### Retoma del proyecto y cierre de `actores`/`efectivo` (5 de Septiembre, 2026)
Tras varios meses sin actividad, se retomó el proyecto con una auditoría completa
del estado real (no solo de lo que decían los `.md`): repos `backend`/`frontend`
limpios y con remoto, 15 tests backend pasando, build de Angular en producción
sin errores. Se cerraron las brechas de proceso (proyecto registrado en el
`CLAUDE.md` del workspace, `docs/` versionado en un repo nuevo `fundacionRebuild`,
esqueleto de `docs/plan-corte.md`) y luego las 2 tablas que quedaban "Pendiente"
en `consolidado.md`:
- **`efectivo`**: investigado antes de preguntar — 13.750 filas pero sin
  movimientos desde 2020-05-22, y el propio legado (`fundacion/tienda/efectivo.php`)
  ya lee de `efectivorobert`, no de esta tabla. Decisión de Luis: dato histórico
  muerto, no se migra ni se construye pantalla nueva.
- **`actores`**: investigado antes de preguntar — catálogo de 1.552 nombres sin
  ninguna FK real en el esquema, usado solo como autocompletado en el legado
  (`ahorro.php`). Al implementar, se encontró que **ya existía** un endpoint
  (`ContabilidadController::getActores`, usado por el módulo Contabilidad) sin
  tests — se descubrió por un test que fallaba tras crear sin querer un endpoint
  duplicado. Se reconcilió reutilizando ese endpoint existente (patrón precarga +
  filtro en cliente, igual que Contabilidad) en vez de duplicar lógica, se le
  agregó cobertura de tests, y se extendió el mismo autocompletado al módulo
  Ahorro (`ahorro.ts`/`ahorro.html`), que antes solo sugería residentes.
  **Lección**: buscar código existente antes de crear uno nuevo — la regla ya
  estaba en el `CLAUDE.md` del workspace, esta vez costó un endpoint duplicado
  y un test roto detectarlo a tiempo.

### Estabilización y Compilación (17 de Mayo, 2026)
Se realizó una auditoría completa del estado de compilación del sistema, resolviendo múltiples inconsistencias técnicas en el Frontend que impedían la generación del bundle de producción (`npm run build` fallaba):
1. **Rutas e Importaciones**: Se corrigió la importación del componente de login en `app.routes.ts` para que apunte correctamente a la clase `Login` exportada por `login.ts` (unificando el criterio con los demás componentes autoportantes).
2. **Servicio API**: Se implementó el método `patch()` en el servicio `Api` (requerido para cambios de estado de residentes y citas de agenda), junto con las propiedades `baseUrl` y `getBaseUrl()` que faltaban para interactuar dinámicamente con los formatos PDF y códigos de barras.
3. **Servicio Auth**: Se añadió el método `getToken()` requerido para la autorización por Query Param en la apertura de PDFs en ventanas nuevas.
4. **Componente Pagos**: Se importó `Router` en `pagos.ts` y se corrigió una etiqueta `</div>` huérfana en `pagos.html` que rompía la estructura jerárquica del HTML (mantenimiento del contenedor `pagos-container`).
5. **Tipado Estricto**: Se resolvió el error de tipo implícito `any` en los manejadores de errores de `dashboard.ts`.

Tras estas correcciones, **el frontend compila al 100% de manera exitosa** y la comunicación API-Sanctum funciona fluidamente con el backend en Laravel 11.

### Estabilización de Reportes Analíticos (18 de Mayo, 2026)
Se integró y estabilizó por completo el nuevo Centro de Reportes:
1. **Prevención de Bloqueos (Loading eterno):** Se corrigieron errores de inicialización y typos en la plantilla Angular `reportes.html` (validación de arrays y safe-navigation operator `?`).
2. **Soporte Multitema (Modo Claro/Oscuro):** Se implementaron anulaciones de contraste globales en `styles.css` para asegurar la legibilidad del 100% de los números, KPIs y textos sobre fondos claros.
3. **Consistencia de Estados:** Se unificó el cálculo de residentes activos a nivel backend y frontend para contar conjuntamente los estados `'A'` (Activo) y `'E'` (Egreso/Especial).
4. **Paginación y Estado Angular:** Se introdujeron controladores de paginación locales e independientes por pestaña para mejorar el rendimiento de renderizado, y se implementó la escucha manual de estados mediante `ChangeDetectorRef.detectChanges()` en los callbacks asíncronos HTTP RxJS.
5. **Modales Interactivos Detallados:** Se diseñaron modales glassmorphic interactivos y scrollables para visualizar los motivos de reingreso y los detalles integrales de evolución de Psicología (resumen, diagnóstico, técnicas y tareas).

### Retoma de sesión: cierre de 2 bloqueantes de `plan-corte.md` (5 de Septiembre, 2026, tarde)
Continuación de la auditoría de la mañana. Se resolvieron, sin necesitar decisión
de negocio, dos de los ítems de la sección 3 ("Validación previa al corte"):
- **Cobertura de tests de Ingreso**: no existía ningún test para el proceso
  multi-tabla atómico más crítico del sistema. Se creó
  `backend/tests/Feature/IngresoTest.php` (6 tests, 22/22 pasan en la suite
  completa) cubriendo: creación atómica cross-tabla de un residente nuevo
  (`residentes`, `actores`, `cobrospension`+`abonopensiones`, `uniformes`,
  `historiali`, `historial`, `historialm`, `usuarios`+`asociacion` para el
  acudiente), reenvío del mismo documento sin duplicar cargos/historiales,
  acudiente que ya es staff conserva su rol (soporte multi-rol), y rollback
  transaccional completo si falta `guardian_data`. Antes de escribir el
  rollback se verificó en `ResidenteController::updateStatus` que el patrón
  "si ya existe, no duplicar" de `historiali`/`historial`/`historialm` en
  `IngresoController` es correcto y no un bug: el reingreso real (residente
  Inactivo que vuelve a Activo) pasa por `updateStatus`, no por `/ingresos` de
  nuevo — evitó una corrección innecesaria sobre código que ya funcionaba bien.
  Queda pendiente cobertura de biometría (firma/huella) dentro de Ingreso.
- **Mecanismo MD5 → Bcrypt**: confirmado en `AuthController::login` — intenta
  Bcrypt primero, cae a MD5 contra `validacion.password` (ahí vivía el hash
  legacy real; `usuarios.password` viene vacío en el dump), con upgrade
  silencioso a Bcrypt al validar por esa vía. Transparente para el usuario.
  Se corrigió `consolidado.md`: la fila de `validacion` describía solo su
  función de auditoría biométrica, no que también es el repositorio del
  password legacy — la tabla cumple doble función.

Sigue abierto (requiere decisión de negocio de Luis, no se resuelve solo):
estrategia de corte a producción y estado real de operación del legado —
secciones 1 y 2 de `plan-corte.md`.

### Daily de plan-corte: las 3 decisiones de negocio cerradas (5 de Septiembre, 2026, tarde)
Daily puntual con Luis enfocado exclusivamente en las preguntas abiertas de
`plan-corte.md`. Resultado — las 3 secciones de decisión de negocio quedaron
cerradas en la misma sesión:
- **§ 1 (operación actual)**: el legado sigue operando activo hoy, en el
  mismo hosting de origen del dump (cPanel/Hostinger), con acceso propio de
  Luis y **solo backups manuales** (sin automatización). Esto expuso un
  detalle técnico no evidente hasta ahora: `backend/.env` apunta hoy a MySQL
  **local** (copia del dump), no a la base real — sea cual sea la estrategia,
  hay que apuntar el sistema nuevo a la base real en algún momento para no
  perder lo que el legado sigue generando.
- **§ 2 (estrategia de corte)**: **Big Bang**. Se descartaron "por módulo" y
  "piloto por sede" porque ambas exigían resolver la convivencia de los dos
  sistemas escribiendo en paralelo sobre la base real — costo que Big Bang
  evita. Esto simplificó la checklist de § 3: ya no hace falta esa
  convivencia, pero subió de prioridad el staging con una copia **fresca**
  de la base real (no el dump de 8 años) y una ronda de QA formal que cubra
  **todos** los módulos antes del corte (no puede quedar ninguno para
  después, como sí permitía el corte por módulo).
- **§ 4 (rollback)**: legado disponible en modo solo-lectura **1 semana**
  post-corte; la decisión de activar un rollback la toma Luis, sin un
  criterio formal pre-establecido.

Lo único que queda pendiente en `plan-corte.md` es la checklist técnica de
§ 3 (staging, QA formal completa, backup manual explícito pre-corte, fecha de
la ventana) — sin más bloqueantes de negocio de por medio.

### Ronda de QA formal de § 3: 2 bugs reales encontrados en el primer batch (5 de Septiembre, 2026, noche)
Arranque de la ronda de QA formal de `plan-corte.md` § 3, por orden de riesgo
(módulos que mueven dinero real primero, ya que con Big Bang decidido no hay
coexistencia con el legado que amortigüe un error). Cobertura agregada:
`PagoTest.php`, `ChargePensionsTest.php`, `TiendaTest.php`, `AhorroTest.php`
(25 tests nuevos, 47/47 pasan en la suite completa). Se encontraron y
corrigieron 2 bugs de correctness reales en código ya dado por "implementado"
en `consolidado.md` — exactamente el tipo de hallazgo que esta ronda existe
para atrapar antes del corte, no después:

1. **`pensions:charge` crasheaba siempre.** El comando programado
   (`Schedule::command('pensions:charge')->dailyAt('01:00')`, ver
   `routes/console.php`) usa `CobroPension::whereHas('residente', ...)`, pero
   el modelo `CobroPension` no tenía ningún método `residente()` definido —
   `BadMethodCallException` en cada ejecución, confirmado corriendo el
   comando dentro del harness de test (no en tinker, que apunta a MySQL real
   inexistente en esta máquina). Si esto llegaba a producción tal cual, el
   cobro automático de pensiones fallaba en silencio todas las noches, sin
   ningún mecanismo que lo detectara hasta una conciliación financiera. Fix:
   se agregó la relación (mismo patrón que ya usa `Uniforme::residente()`).
2. **`TiendaController::storeSale` no validaba stock en el servidor.**
   `consolidado.md` documenta "validar stock contra la base de datos" como
   parte ya implementada del POS, pero el endpoint insertaba la venta y
   descontaba el saldo del residente sin chequear en ningún punto que
   hubiera unidades disponibles — se podía vender en stock negativo
   llamando al endpoint directo. Fix: se agregó la validación (atómica por
   carrito — si un solo ítem no tiene stock suficiente, se rechaza la venta
   completa antes de tocar la base) más los tests de regresión.

Quedó 1 observación, no corregida a propósito por no tener base documental
que la respalde (a diferencia de los 2 casos de arriba): en
`AhorroController::store`, una `salida` (retiro) no valida contra el
acumulado disponible — puede dejar el saldo general en negativo. A
diferencia del caso de Tienda, no había ninguna mención en `consolidado.md`
de que esa validación debiera existir, así que no se asumió una regla de
negocio nueva sin confirmar con Luis primero. **Decisión de Luis
(2026-09-06): se deja como está, es comportamiento intencional** — cerrado,
sin cambio de código.

### Cierre de la ronda de QA formal: 24/24 controllers, 6 bugs reales en total (5 de Septiembre, 2026, madrugada)
Continuación y cierre de la ronda de QA formal de `plan-corte.md` § 3.
147 tests nuevos, 153/153 pasan en la suite completa. 3 bugs más
encontrados y corregidos en este tramo (Agenda x2, Reporte x1), sumando
**6 en total** en toda la ronda — todos con el mismo patrón de fondo:
código escrito contra una columna/relación que nunca existió en el
esquema real, invisible hasta correr un test contra el schema correcto
(SQLite no valida columnas inexistentes en un WHERE de la misma forma
que MySQL, así que varios de estos quedaban enmascarados incluso
escribiendo tests, hasta construir el schema del test fiel al dump):

4. **`agenda.encargado` no existe.** Ni en el esquema legado ni en
   ninguna migración posterior — `POST /api/agenda` y el filtro
   `?encargado=` tiran "Unknown column" siempre. Se agregó la migración
   `2026_09_05_230000_add_encargado_to_agenda_table.php` (agregar
   columnas a tablas legadas está permitido por `CLAUDE.md`).
5. **`Agenda` sin relación `residente()`.** `AgendaController::index()`
   hace `Agenda::with('residente')` incondicionalmente — `GET /api/agenda`
   crashea siempre, con o sin filtros, con `RelationNotFoundException`.
   Se agregó la relación.
6. **`ReporteController::inventory()` con columnas inventadas.**
   Seleccionaba `nombre`/`stock`/`precio` de `productos` — ninguna existe
   (reales: `detalle`/`valorcompra`/`valorventa`, `stock` es calculado).
   Mismo patrón exacto que el bug de `TiendaController::storeSale`
   encontrado antes en la misma ronda. Se corrigió reusando el cálculo
   de stock ya existente en `TiendaController::inventory()`.

Dos endpoints quedan sin test unitario por una limitación real de
motor, no por falta de cobertura: `SystemController::getStatus()`
(`SUBSTRING_INDEX(GROUP_CONCAT(...))`) e `incomeByMonth()`/la porción
"por_mes" de `seguimientosPsicologia()` (`DATE_FORMAT`) usan sintaxis
específica de MySQL que SQLite no soporta — documentado en
`SystemTest.php`/`ReporteTest.php`, pendiente de validación manual
contra MySQL real cuando exista el ambiente de staging.

También se encontró (`BitacoraController`) un comportamiento real no
obvio, no un bug: cada request a `/api/bitacora` se audita a sí misma
vía `ApiBitacoraMiddleware` (aplicado globalmente a toda la API) — un
test que hace 2 llamadas seguidas al mismo endpoint dentro del mismo
método ve contaminada la segunda por el log de la primera. Los tests
de `BitacoraTest.php` filtran por criterios que esa auto-auditoría
nunca produce, en vez de contar filas totales.

Con esto, `plan-corte.md` § 3 queda completo salvo 3 ítems que ya no
son responsabilidad de esta ronda: staging con copia fresca de la base
real, backup manual pre-corte, y fecha/horario de la ventana — los tres
dependen de acción/decisión de Luis, no de más trabajo de QA.

### Fix real del bloqueo de "modo autónomo" en este proyecto (5 de Septiembre, 2026, noche)
Incidente largo en medio de la ronda de QA: "modo autónomo" (mecanismo
workspace-wide, ver `CLAUDE.md` de Softclass) no tenía ningún efecto acá,
pese a funcionar bien en el resto de los proyectos del workspace, en la
misma máquina. Causa raíz real, confirmada contra la documentación oficial
de Claude Code (`github.com/anthropics/claude-code` issue #12962): Claude
Code no sube a directorios padre a buscar `.claude/settings.json` cuando el
directorio de trabajo actual tiene su propio `.git` — y `backend/`/`frontend/`
son justo eso, repos propios (decisión deliberada de este proyecto, ver
`CLAUDE.md`). Como ninguno tenía su propio `.claude/settings.json`, los hooks
de `risk-classifier.sh`/`mode-marker-write.sh` (que viven en la raíz,
`FundacionRebuild/.claude/settings.json`, un repo distinto) nunca se
ejecutaban ahí. Fix: copiar el mismo `settings.json` a `backend/.claude/` y
`frontend/.claude/`, commiteado en cada repo respectivo.

Nota aparte para no repetir la confusión: durante el diagnóstico aparecieron
bloqueos de "Blocked by classifier" al intentar editar `risk-classifier.sh` y
escribir `settings.json` — eso es una capa totalmente distinta (el
clasificador de seguridad propio de Claude Code, no configurable por hooks ni
por proyecto), que bloquea a cualquier agente editando infraestructura de
permisos en cualquier proyecto, en cualquier máquina. No es un bug de este
proyecto ni tiene fix posible — es intencional.

### Validación manual Angular ↔ backend contra datos reales: 1 bug real de severidad alta (6 de Septiembre, 2026)
Luis pidió parar de asumir "cubierto y funcional" solo por la ronda de QA de
backend (ver entrada anterior) y validar de verdad con Angular corriendo
contra el backend local — la pregunta correcta, dado que toda la ronda
anterior nunca había tocado el frontend ni datos reales.

**Ambiente levantado desde cero** (no existía nada montado):
- Contenedor Docker **dedicado** `fundacion_db` (MySQL 8, puerto **3307**) —
  el puerto 3306 estándar ya estaba ocupado por `factoring_db` (Proseguir),
  otro proyecto corriendo en la misma máquina. Se optó por no tocar ese
  contenedor ajeno.
- Dump legado real importado ahí (`fundacion/u727327027_fjemr.sql`, 17MB):
  **742 residentes, 777 usuarios reales**, 52 tablas.
- `php artisan migrate` corrió limpio sobre ese esquema real — confirma que
  todas las migraciones de esta sesión (incluida `add_encargado_to_agenda`
  de hoy) aplican bien contra datos de producción, no solo contra el
  schema sintético de los tests.
- Backend (`php artisan serve`) en **puerto 8010** — el 8000 estándar
  también estaba ocupado, esta vez por `factoring_backend_web` (Docker,
  nginx). Un `curl` a `localhost:8000/api/login` devolvía una respuesta con
  campos de OTRO proyecto (`numero_documento`) antes de notar el conflicto.
- Frontend (`ng serve`) en puerto 4200, `environment.ts` apuntado
  temporalmente al backend real durante la prueba (revertido a 8000 antes
  de commitear — el puerto 8010 es un workaround de esta máquina, no algo
  para fijar en el repo).
- Playwright instalado como devDependency real del frontend (antes no
  existía) — regla dura del workspace: toda validación visual/funcional en
  navegador va por Playwright CLI, nunca por la extensión Claude in Chrome
  (desinstalada). Queda disponible para próximas validaciones.

**Bug real encontrado, severidad alta** (no lo hubiera atrapado ninguna
ronda de tests de backend — es puramente de frontend): `Api.handleApiError()`
redirigía la página completa (`window.location.href = '/login'`, borrando
`localStorage`) ante **cualquier** 401 — incluido el 401 normal de un login
con contraseña incorrecta. Un usuario real escribiendo mal su clave nunca
veía "Credenciales inválidas": la pantalla se recargaba en blanco antes de
que el mensaje llegara a mostrarse. Confirmado paso a paso con Playwright
(estado interno del componente Angular vía `window.ng.getComponent`, no solo
capturas): campos y `error` se reseteaban a vacío porque el login entero se
recargaba desde cero. Fix: el redirect por sesión expirada ahora solo
dispara si ya había un token guardado (`localStorage.getItem('token')`) —
un intento de login fallido nunca tiene token todavía, así que deja de
disparar el redirect y el mensaje de error se muestra normal. Verificado
visualmente tras el fix: banner "⚠️ Credenciales inválidas" visible, campos
conservan lo escrito, botón vuelve a habilitarse.

Se aprovechó para agregar también `ChangeDetectorRef.detectChanges()` en
`Login.onLogin()` (no era la causa raíz de este bug puntual, pero cerraba
el mismo gap que el resto de la app ya resolvió — ver sesión 18-May más
arriba). Al revisar esto se encontró un **hallazgo más amplio, sin corregir
esta sesión**: de 24 componentes que usan `HttpClient.subscribe()`, solo 7
ya inyectan `ChangeDetectorRef`, pese a que `memory.md` documenta esto como
patrón obligatorio bajo `provideZonelessChangeDetection()` desde mayo.
Quedan **17 componentes con el mismo riesgo potencial** (estado que no se
refleja en pantalla tras una respuesta async) — no se tocaron a ciegas,
queda como ítem propio en `plan-corte.md` § 3 para una auditoría dedicada.

Después de esto, con login real funcionando de punta a punta contra datos
reales (63 residentes activos visibles en el dashboard, nombres/documentos
reales, paginación correcta), el build de producción de Angular (`ng build
--configuration production`) sigue compilando limpio — solo warnings
preexistentes sin relación (presupuesto de CSS de terapias, `sweetalert2`
no-ESM).

**Conclusión honesta para Luis** (ver también la respuesta dada en el chat):
esto NO significa "refactoring cubierto y funcional" en el sentido de listo
para producción — significa que el camino de login, antes roto en silencio,
ahora funciona de verdad, y que hay una categoría entera de bugs de frontend
(estado que no se actualiza bajo zoneless CD) todavía sin auditar en 17
componentes más.

## Módulos Implementados

### 1. Núcleo Administrativo y Seguridad
- **Autenticación**: Login seguro por documento con tokens Sanctum.
- **Identidad Corporativa**: Integración de logos y paleta de colores de JOREC y Jesús es mi Roca en toda la interfaz.
- **Bitácora (Audit Log)**: Registro inmutable de cada acción (quién, qué, cuándo, IP) para auditoría.
- **Documentación**: Swagger integrado para todos los endpoints de la API.

### 2. Gestión de Residentes (Ciclo de Vida)
- **Ingreso Multi-tabla**: Registro atómico que sincroniza 8 tablas del esquema legado.
- **Biometría**: Captura de firmas en tiempo real (tabletas/mouse) y carga de huellas dactilares.
- **Gestión de Estados**: Flujo completo de Inactivación (con motivo y fecha) y Reactivación (con ajuste de pensión).
- **Historial Consolidado**: Vista 360° de cada residente (atenciones, pagos, estancias).

### 3. Finanzas y Pagos
- **Control de Pensiones**: Listado inteligente con saldos y estados de cuenta.
- **Abonos y Cargos**: Registro de movimientos financieros con soporte para comentarios y fechas personalizadas.
- **Automatización**: Tarea programada (`pensions:charge`) que genera cobros diarios automáticamente.

### 4. Psicología y Seguimiento
- **Módulo Clínico**: Registro detallado de atenciones con campos de resumen, evaluación, técnicas y tareas.
- **Timeline**: Visualización cronológica de la evolución del paciente.
- **Reportes PDF**: Generación de informes de atención individual con firmas biométricas.

### 5. Agenda y Citas
- **Control de Consultorio**: Programación de citas para psicología y medicina.
- **Gestión de Asistencia**: Estados de cita (Espera, Atendido, No asistió) con trazabilidad.

### 6. Tienda e Inventario (POS)
- **Punto de Venta**: Venta de productos con descuento automático del saldo del residente.
- **Códigos de Barras**: Generación y lectura de etiquetas Code128 para productos y carnets de residentes.

### 7. Formatos de Impresión (Alta Fidelidad)
- **Documentos Legales**: Formulario de ingreso dinámico en PDF.
- **Identificación**: Carnets individuales con código de barras y foto.
- **Listados Masivos**: Hojas de códigos de barras para inventario y control de residentes.

### 8. Centro de Reportes Analíticos
- **KPIs en Tiempo Real**: Panel de control con estado general del establecimiento (pensiones, activos, psicología, conceptos, salidas).
- **Control de Reingresos**: Historial de estancias con búsqueda y modales interactivos para motivos.
- **Evolución Psicológica Detallada**: Muestra las fichas clínicas completas, técnicas aplicadas y tareas mediante un modal detallado, manteniendo limpia la tabla principal.
- **Control de Salidas (Permisos)**: Control dinámico para residentes en estado Fuera y Retornado.

## Arquitectura Técnica
- **Frontend**: Angular 19 (Standalone Components, RxJS, Glassmorphism CSS).
- **Backend**: Laravel 11 (Controllers, API Resources, Observers).
- **Base de Datos**: MariaDB Legada (Preservada y sincronizada).
- **Offline**: OfflineManager con IndexedDB para resiliencia en red inestable.

## Conclusión del Día
Se han cubierto todos los requerimientos críticos de modernización, entregando una plataforma rápida, segura y estéticamente superior a la original.
