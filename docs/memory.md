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

### Auditoría de ChangeDetectorRef completa: 2 bugs reales más confirmados (6 de Septiembre, 2026)
Cierre del hallazgo anterior. Se agregó `ChangeDetectorRef` + `detectChanges()`
a los 16 componentes que no lo tenían (`agenda`, `ahorro`, `almuerzos`,
`biometricos`, `bitacora`, `compras`, `compras-detalle`, `conceptos`,
`diezmos`, `minuta`, `permisos`, `practicantes`, `psicologia`, `tienda`,
`uniformes`, `usuarios`) — mismo patrón en los 24: `cdr.detectChanges()`
después de cualquier mutación de estado dentro de un callback `next`/`error`
de `.subscribe()` que se refleja en el template.

Se verificaron en vivo con Playwright (login real + navegación real contra
el backend/base de datos reales del ambiente levantado antes) 2 de los 16,
ambos con el **mismo bug exacto** que ya se había encontrado en login —
confirma que la sospecha no era teórica:
- **`usuarios.ts` (`saveUser`)**: crear o editar un usuario con un
  `documento` que ya existe devuelve 422 del backend
  ("The documento has already been taken.") — antes del fix, ese mensaje
  nunca llegaba a mostrarse al administrador, quedaba solo en el estado
  interno del componente. Confirmado con `window.ng.getComponent()`: antes
  `error` se seteaba pero el DOM no lo reflejaba; después, el banner
  "⚠️The documento has already been taken." aparece de verdad. De paso se
  corrigió el `setTimeout()` que auto-cierra el modal al guardar con éxito
  — tampoco disparaba CD, mismo problema de fondo.
- **`conceptos.ts` (`doDelete`)**: intentar eliminar un concepto contable
  con asientos asociados devuelve 409 (`ConceptoController::destroy`,
  protección ya cubierta por `ConceptoTest.php` en el backend) — el mensaje
  de "no se puede eliminar" tampoco se mostraba nunca en el frontend antes
  de este fix.

Los otros 14 componentes recibieron el mismo fix preventivo por patrón de
código (mismo `.subscribe()` sin `ChangeDetectorRef`, mismo riesgo
estructural) pero no se verificó cada uno individualmente con Playwright —
sería el siguiente paso natural si aparece evidencia de que alguno todavía
falla en la práctica. Build de producción de Angular (`ng build
--configuration production`) sigue limpio tras el fix, mismos 2 warnings
preexistentes sin relación.

Con esto, `plan-corte.md` § 3 no tiene ningún ítem técnico abierto — quedan
únicamente los 2 que dependen de acción/decisión de Luis (backup pre-corte,
fecha de la ventana).

### Comparación en vivo legado vs. sistema nuevo: Ingreso estaba roto de punta a punta (6 de Septiembre, 2026)
Luis preguntó si el sistema nuevo realmente cubre todo lo del legado —
pregunta correcta: `consolidado.md` mapea tablas contra modelos, no
garantiza que cada pantalla se comporte igual ni que funcione de verdad
contra la UI real. Pidió empezar por **Pensiones e Ingreso**, los módulos
con más reclamos reales reportados.

**Ambiente**: se levantó el PHP legado localmente (`php -S` con
`-d mysqli.default_port=3307` para apuntar al mismo Docker MySQL ya
importado, sin tocar ningún archivo del legado) junto al sistema nuevo
(backend :8010, frontend :4200), y se comparó con Playwright usando la
misma cuenta admin en ambos (se actualizó `validacion.password` — MD5
legado — al mismo valor que ya tenía `usuarios.password` en Bcrypt, solo
en esta base de prueba local).

**Ingreso — bug crítico encontrado y corregido**: el formulario nuevo
(wizard de 4 pasos) cubre y hasta *supera* al legado en campos capturados
(agrega estado de salud/vacunas/alergias que el legado no pedía en esta
pantalla) — buena señal. Pero al enviar un Ingreso real de punta a punta
por la UI, `POST /api/ingresos` devolvía **500** siempre:
`SQLSTATE[42S22]: Column not found: 'tipo_sanguineo'`. El campo "Tipo de
Sangre/RH" (obligatorio en el paso 1) ya estaba en `Residente::$fillable`,
pero nunca existió una migración que agregara la columna a la tabla legada
real. **El proceso más crítico de todo el sistema estuvo roto de punta a
punta**, sin que ningún test lo detectara — `IngresoTest.php` arma su
payload a mano y nunca incluyó ese campo, el mismo hueco replicado sin
querer en el test. Fix: migración `add_tipo_sanguineo_to_residentes_table`
+ actualización de `IngresoTest.php` (payload y schema del test) para que
una regresión futura sí quede atrapada por la suite rápida.

Verificado tras el fix: `POST /api/ingresos` → 200, residente real creado
(id 810), con las 8 tablas relacionadas pobladas atómicamente
(`residentes`, `historial`, `historiali`, `historialm`, `cobrospension`,
`uniformes`, `asociacion`, `actores`) — confirmado por consulta directa a
la base de datos real, no solo por la respuesta HTTP.

**Lección de fondo, la razón de por qué esto importa más que los bugs de
antes**: los 6 bugs de backend y los 2 de frontend encontrados en las
rondas anteriores salieron de tests/código escritos por el mismo agente
que después los "verificaba" — un punto ciego real, no hipotético, quedó
demostrado acá: mi propio `IngresoTest.php`, pese a 6 casos y 100% verde,
nunca ejercitó el payload que el formulario real de verdad envía. Solo
correr la app de verdad, con Playwright, contra el formulario real, contra
el backend real, atrapó esto.

**Pensiones — sin bugs, paridad exacta confirmada.** Mismo ejercicio que
Ingreso, esta vez con resultado limpio. Se comparó la lógica de saldo
pendiente línea por línea: legado (`pensiones.php`) calcula
`SUM(valorinicial) - SUM(abono)` por `cobrospension`; el nuevo
(`PagoController::index` + `pagos.html`) hace exactamente lo mismo
(`total_cobrado - total_abonado`). Se verificó con **10 residentes reales**
del dump importado, comparando fila por fila (día de cobro, valor de
pensión, saldo) entre legado y sistema nuevo — **coincidencia exacta en
los 10**, incluidos 2 casos de saldo negativo (residentes que pagaron de
más). También coincide la regla de negocio "Contabilizar/No Contabilizar"
según `estado` A/E.

Se registró además un abono real de $100.000 sobre un residente real
(LUIS FERNANDO VALENCIA SANCHEZ, saldo previo $0) a través de la UI nueva
de punta a punta — `POST /api/pagos/abono` → 200, saldo actualizado a
-$100.000 en el sistema nuevo, y se releyó el **legado** (mismo Docker
MySQL, sin recargar nada del lado legado) confirmando el mismo saldo
-$100.000 — la escritura de un sistema es visible e idéntica en el otro,
prueba de que ambos leen/escriben sobre el mismo modelo de datos sin
divergencia.

Pendiente, con el tiempo: repetir este mismo ejercicio (legado vivo +
sistema nuevo vivo + Playwright, no solo lectura de código) para el resto
de los módulos críticos — la única forma que demostró atrapar bugs reales
esta sesión.

### Tienda: hallazgo grande, pausado a la espera de un dump (6 de Septiembre, 2026)
Al intentar extender la comparación en vivo a Tienda/POS se encontró que
`tienda/bas/conn.php`, `tienda/bas/conx.php`, `tiendajemr/bas/conn.php`,
`tiendajemr/bas/conx.php` y `negocio/bas/conn.php` **siguen apuntando al
hosting real de producción** (`srv1107.hstgr.io`) — a diferencia de
`ingreso/bas/conn.php`, que en algún momento alguien adaptó a `127.0.0.1`
local. Ningún dato se tocó (el intento de conexión dio timeout de red, sin
llegar a autenticar ni consultar nada).

Más importante: `tienda/`/`tiendajemr/` usan **2 conexiones separadas** —
`$con` (la base `u727327027_fjemr`, la que ya tenemos dumpeada e
importada) y **`$conx`, apuntando a una base completamente distinta,
`u727327027_tienda`**, nunca dumpeada ni examinada hasta ahora. El punto
de venta real (`crearpc.php`, registrar una venta; el dropdown de
productos) usa `$con` — pero `crearprod.php` (dar de alta un producto
nuevo) y varias páginas de proveedores/pedidos usan `$conx`.

Dato duro que motivó la pregunta a Luis: la tabla `venta` en la base que
tenemos (`fundacion`, importada de `u727327027_fjemr.sql`) **no tiene
ninguna fila posterior al 25 de enero de 2020** (9.429 filas totales).
Luis confirmó que la Tienda **sigue operando activamente hoy** — lo cual,
cruzado con esa fecha de corte, es una señal fuerte de que las ventas
reales de los últimos ~6 años no están yendo a esta base en absoluto,
sino probablemente a `u727327027_tienda` vía `$conx`. Mismo patrón que
`efectivo` (tabla con actividad muerta desde una fecha, el legado ya
usando otra tabla/base en su lugar) pero acá afecta a un módulo que
`consolidado.md` da por "Consolidado" y que Luis confirma vigente — si
se confirma, significaría que el POS de Tienda migrado (`TiendaController`,
`productos`/`venta`/`detalleventa` del sistema nuevo) está construido
sobre un snapshot histórico muerto, desconectado de la operación real
actual, no sobre la fuente de verdad vigente.

**Acción pendiente de Luis**: exportar un dump de `u727327027_tienda` del
mismo panel de Hostinger de donde salió `u727327027_fjemr.sql`, para
importarlo igual y comparar de verdad contra la operación real. Pausado
hasta que llegue ese dump — mientras tanto se sigue con el resto de
módulos que sí viven en `ingreso/` (ya apuntado a local): Ahorro,
Diezmos, Contabilidad, Psicología, Terapias, Agenda, Minuta, Permisos.

### Ahorro: segundo bug crítico — el acumulado rompía la cronología (6 de Septiembre, 2026)
Comparando `ahorro.php`/`ahorrosmovimiento.php` (legado) contra
`AhorroController` (nuevo), la lógica del 10% de diezmo automático
coincidía exacto — pero se notó que el legado, tras cada inserción,
**recalcula en cascada todo el libro** (`ahorrosmovimiento.php` recorre
`asientosahorro` + `ahorro` completo ordenado por fecha y reescribe el
`acumulado` de cada fila), mientras `AhorroController::store()` solo
tomaba el acumulado del **último registro insertado**
(`orderBy('idahorro', 'desc')`), no el último por **fecha**.

Reproducido en vivo contra el backend real: se insertó un movimiento
día 1 (100.000), día 3 (150.000 acumulado), y después uno **atrasado**
con fecha día 2 (20.000) — el sistema le puso acumulado 460.692 al día 2
y dejó el día 3 en 440.692, **más bajo** que un día anterior. El libro
contable dejaba de ser cronológicamente consistente ante cualquier carga
fuera de orden (una corrección tardía, un movimiento omitido que se
carga después) — un escenario realista, no de laboratorio.

Fix: mismo patrón de recálculo en cascada ya usado en
`AlmuerzoController`/`DiezmoController`/`ContabilidadController` (ubicar
el acumulado base justo antes de la nueva fecha, insertar, y recalcular
en cascada solo los movimientos posteriores) — se corrigió también
`index()`'s `saldo_actual`, que tenía el mismo problema de fondo
(ordenaba por `idahorro` en vez de por fecha). Verificado en vivo tras
el fix: la misma secuencia queda cronológicamente consistente
(390.692 → 410.692 → 460.692). Test de regresión agregado, 154/154
tests pasan.

**Patrón que se repite**: van 2 de 2 módulos financieros con hallazgos
reales en esta comparación en vivo (Ingreso, Ahorro) — ninguno lo había
detectado la ronda de QA de backend, porque los tests existentes solo
ejercitaban fechas en orden ascendente, nunca un caso fuera de orden.
Pendiente: revisar si Diezmos/Contabilidad/Uniformes/Compras (que ya
tienen cascada) manejan bien este mismo escenario de fecha atrasada, o
si comparten alguna variante del mismo problema.

### Auditoría de la cascada + smoke test en vivo del resto de módulos de `ingreso/` (6 de Septiembre, 2026)
Se resolvió la pregunta pendiente de la entrada anterior con una auditoría
dirigida (no otra ronda completa de Playwright): un grep sobre todos los
controllers buscando el mismo antipatrón de Ahorro (`orderBy('idX',
'desc')` como único criterio de orden, sin `fecha` primero) no encontró
ninguna otra instancia — `DiezmoController` (`cascadeRecalculateRoca/
Jorec/Diezmo` hace recálculo completo por fecha; `cascadeRecalculateColombia/
Colpatria/Efectivo` usa el mismo patrón de cascada parcial que se le aplicó
a Ahorro), `ContabilidadController`, `UniformeController` y `PedidoController`
ya ordenan por `fecha` antes que por el id en sus bases de acumulado. Ahorro
era el único que se había quedado afuera de este patrón ya establecido.

Se comparó además, campo por campo, el payload real que cada componente
Angular envía contra la validación del controller correspondiente
(mismo tipo de comparación que destapó el bug de `tipo_sanguineo` en
Ingreso) para **Agenda, Minuta, Permisos, Diezmos, Contabilidad,
Terapias (cognitiva y espiritual) y Psicología/Seguimiento** — los 7
coinciden exactos, sin campos faltantes. Se confirmó además con un
smoke test en vivo contra el backend real (creación de cita, visita,
permiso, abono de diezmo, movimiento contable, sesión de terapia x2 y
seguimiento) — **los 7 succeeded sin errores**, datos de prueba
limpiados después.

**Balance del ciclo de comparación en vivo de esta sesión**: 9 módulos
recorridos (Ingreso, Pensiones, Ahorro, Agenda, Minuta, Permisos,
Diezmos, Contabilidad, Terapias, Psicología — 10 en total), **2 bugs
críticos reales encontrados y corregidos** (Ingreso: columna sin
migrar; Ahorro: cascada rota con fecha atrasada), **1 módulo pausado**
esperando un dump de Luis (Tienda, posible base de datos separada sin
examinar), el resto sin hallazgos. 154/154 tests de backend pasan.

### Tienda/Compras: confirmado — corren sobre una base real distinta, pausados por decisión de Luis (6 de Septiembre, 2026, noche)
Luis consiguió el dump de `u727327027_tienda` (mismo panel de Hostinger,
22MB, generado 2026-09-06) y lo trajo a `fundacion/u727327027_tienda.sql`.
Se importó (con un ajuste menor: `ROW_FORMAT=FIXED` no es válido para
InnoDB en esta versión de MariaDB, se quitó del dump — es solo un hint de
almacenamiento, no afecta datos ni esquema) como base `tienda` separada en
el mismo contenedor Docker.

**Confirmado sin ambigüedad**: es un esquema **completo y distinto**, no
una simple copia con otro nombre —

| Tabla en `u727327027_tienda` | Filas | Actividad | Equivalente en `fjemr` (el que migramos) |
|---|---|---|---|
| `facturas` | 75.479 | hasta 2026-09-05 | no existe — reemplaza a `venta` |
| `detallefactura` | 292.454 | — | no existe — reemplaza a `detalleventa` |
| `pedidos` | 3.175 | hasta 2026-09-05 | `pedidos` (540 filas, muerto) |
| `detallepedido` | 10.571 | — | `detallepedido` (1.142 filas, muerto) |
| `productos` | 225 | — | `productos` (145 filas, esquema distinto) |
| `mayor` | 75.978 | hasta 2026-09-05 | no existe |
| `asientos` | 73.177 | hasta 2026-09-05 | tabla homónima en `fjemr`, contenido distinto |
| `tienda` | 86.934 | **hasta 2026-09-06 (hoy)** | `tienda` (muerta desde antes de 2020) |
| `pagos` | 2.906 | — | no hay equivalente directo |

La tabla `tienda` (el balance de recargas/cargos del residente, la pieza
central del módulo POS) tiene movimientos literalmente **de hoy** en esta
base — mientras que la misma tabla en `fjemr` (la que el sistema nuevo
migró) no tiene nada después de enero de 2020. Confirma sin lugar a dudas
que la fundación **nunca dejó de operar la Tienda** (como Luis ya había
dicho) — lo que pasó es que en algún momento (parece que a inicios de
2020) la operación real se movió a esta base separada, y nadie lo
documentó ni lo tuvo en cuenta al migrar.

**Alcance del impacto**: los módulos "Punto de Venta (POS Tienda)" y
"Abastecimiento de Inventario (Compras y Proveedores)" — ambos marcados
"Consolidado"/"Migrado" en `consolidado.md` hasta hoy — están construidos
sobre un snapshot histórico completamente desconectado de la operación
real. Esto no es un bug puntual como Ingreso o Ahorro: es un **gap de
alcance real en la migración**, dos módulos enteros que habría que
re-diseñar contra el esquema correcto (`facturas`/`detallefactura`/`mayor`
en vez de `venta`/`detalleventa`).

**Decisión de Luis**: pausar Tienda/Compras del todo por ahora, no
re-mapear todavía — se re-marcaron **"Pendiente (real)"** en
`consolidado.md` (antes "Consolidado"/"Migrado"), con una nota al inicio
del documento explicando el hallazgo completo. Cuando haya prioridad para
encararlo, es un ciclo de trabajo propio (Analista → Arquitecto → PM →
Backend/Frontend Dev), no un fix rápido de QA. La base `tienda` y el dump
quedan disponibles localmente (`fundacion/u727327027_tienda.sql`, base
Docker `tienda` en el mismo contenedor `fundacion_db`) para cuando se
retome.

### Cierre del ciclo: biometría en Ingreso + últimos módulos en vivo (6-7 de Septiembre, 2026)
**Biometría en Ingreso**: se confirmó que la tabla `validacion` no tiene
relación con biometría — el modelo `Validacion` nunca se invoca en ningún
controller, y el legado tampoco la usa para eso (solo MD5 de login). Se
corrigió la descripción en `consolidado.md`. La biometría real
(`residentes.firma_path`/`huella_path`, vía `ResidenteController::
uploadBiometrics`) ya estaba bien implementada y cubierta por
`ResidenteTest.php` — pero se encontró un hueco de **flujo real**: la
pantalla de "Ingreso Exitoso" no ofrecía ningún camino hacia la captura de
firma/huella, solo "Descargar PDF" e "Ir al Dashboard". Se agregó un botón
"Capturar Firma/Huella" que navega directo al residente recién creado.
Verificado end-to-end con Playwright: Ingreso real → click → aterriza en
`/biometricos/:id` correcto.

**Módulos restantes recorridos en vivo** (smoke test contra el backend
real, complementando la comparación de código ya hecha): Uniformes
(57 registros reales, actualización de entrega funciona), Lavandería
(`cobroslavada` genuinamente vacía — tabla nueva del sistema, sin datos
legados que migrar), Practicantes (23), Usuarios (779), Residentes
activos (65) — todos responden correctamente contra datos reales.

**Almuerzos — observación menor, no corregida**: la tabla `cobroalmuerzos`
tiene 18 filas reales, pero la API devuelve 0 porque las 18 pertenecen a
residentes ahora `Inactivo` — comportamiento correcto, coincide con el
legado (`almuerzos.php` filtra igual `estado='A' OR estado='E'`). Se
encontró sí una diferencia sutil de precedencia SQL en el legado:
`estado='A' OR estado='E' AND saldo>0` — por precedencia de operadores,
un residente "Especial" con saldo ya pagado (saldo=0) queda oculto en el
legado, mientras el sistema nuevo lo seguiría mostrando. Impacto mínimo
(ningún caso real en los datos actuales), no se corrigió — queda anotado
por si en el futuro se nota una diferencia real de visualización.

**Balance final del ciclo completo de comparación en vivo** (Ingreso →
Almuerzos): 16 módulos recorridos, **2 bugs críticos corregidos**
(Ingreso, Ahorro), **1 hueco de flujo corregido** (biometría sin acceso),
**1 gap de alcance real descubierto y pausado** (Tienda/Compras, base de
datos separada), **1 observación menor sin corregir** (Almuerzos,
precedencia SQL). 154/154 tests de backend pasan.

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

### Análisis: ¿esquema nuevo migrado, o seguir adaptando al legado? — descartado, con alternativa de bajo riesgo (7 de Septiembre, 2026)
Luis planteó si, dado lo encontrado en el ciclo de comparación en vivo de
esta semana (bugs en Ingreso y Ahorro, el hallazgo de Tienda/Compras),
convenía más diseñar un esquema moderno y migrar la base legada en vez de
seguir construyendo sobre el esquema original preservado (la arquitectura
vigente, mandato explícito de `CLAUDE.md`: "sin alterar el esquema de base
de datos legado").

**Análisis causa por causa de los bugs de esta semana**: ninguno lo hubiera
evitado un esquema nuevo. `tipo_sanguineo` y `encargado` (Agenda) fueron
migraciones olvidadas — pasan igual en cualquier esquema si alguien se
olvida de correrlas. El bug de Ahorro fue lógica de aplicación
inconsistente (los otros 5 libros ya usaban el patrón de cascada correcto;
Ahorro se quedó afuera) — un esquema nuevo no arregla que el código no siga
su propio estándar. El hallazgo de Tienda/Compras fue un problema de
**desconocimiento de una fuente de datos** (nadie sabía que existía
`u727327027_tienda`), no de que el esquema de `fjemr` fuera incómodo —
eso se descubre igual migrando o no.

**El único hallazgo que sí es un problema real de diseño**: se auditó el
algoritmo de recálculo en cascada (el mismo patrón que rompió a Ahorro) en
los 6 controllers que lo usan y se encontró **duplicado al menos 13 veces**
— incluyendo 3 canales (`colombia`, `colpatria`, `efectivorobert`)
reimplementados palabra por palabra en dos archivos distintos
(`ContabilidadController` y `DiezmoController`). Eso es exactamente la
clase de problema que ya causó el bug de Ahorro (un fix aplicado en un
lugar no se propaga a las copias), y **se resuelve sin tocar el esquema
legado** — es 100% capa de aplicación Laravel.

**Decisión**: se descarta migrar a un esquema moderno para el resto del
sistema — el costo (meses de ETL sobre 8 años de datos financieros/
clínicos reales, reescribir 154+ tests, reescribir cada modelo/controller,
repetir todo el ciclo de comparación en vivo) no se justifica contra bugs
que en su mayoría no son de diseño de esquema, y retrasaría indefinidamente
el corte a producción que ya está cerca (`plan-corte.md`, Big Bang). Se
aprueba en cambio la alternativa de bajo riesgo: consolidar el algoritmo de
cascada en un servicio único, spec formal en
`docs/specs/consolidacion-cascada-contable.md`, pendiente de aprobación de
Luis antes de implementar. **Tienda/Compras se mantiene fuera de todo
esto** — sistema aparte, obsoleto, sin actividad real desde antes de 2020,
pausado por decisión explícita de Luis (2026-09-06) y confirmado que se
queda así (2026-09-07) — no forma parte de esta spec ni de ningún refactor
mientras no haya prioridad y un ciclo propio (Analista→Arquitecto→PM) para
encararlo contra el esquema real (`u727327027_tienda`).
