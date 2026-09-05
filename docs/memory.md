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
