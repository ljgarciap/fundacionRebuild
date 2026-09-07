# Consolidación de Procesos y Base de Datos Histórica

Este documento actúa como la **bitácora de control principal** para la modernización del sistema de la **Fundación Rebuild**. Su prioridad absoluta es **garantizar el respeto y la compatibilidad 100% con la base de datos histórica de 8 años de antigüedad** (`u727327027_fjemr.sql`), construyendo la nueva interfaz en Angular 19 y API en Laravel 11 como una capa superior que lee y escribe directamente en el esquema original sin alterarlo.

> ⚠️ **Hallazgo 2026-09-06 — Tienda/Compras corren sobre una base distinta**:
> la operación real y vigente de Tienda (POS) y Compras/Proveedores vive en
> una **segunda base de datos separada** (`u727327027_tienda`, esquema
> propio con `facturas`/`detallefactura`/`mayor`, no `venta`/`detalleventa`),
> activa hasta hoy (75.479 facturas, 292.454 líneas de detalle, `tienda`
> con movimientos al 2026-09-06). Las tablas `productos`/`proveedores`/
> `pedidos`/`detallepedido`/`pagoproveedores`/`tienda`/`venta`/`detalleventa`
> de `u727327027_fjemr` (las que este documento marcaba "Consolidado" más
> abajo) están **desconectadas de esa operación real** — sin actividad
> desde antes de 2020. Los módulos 4 y 9 de la sección 2 de este documento
> quedan re-marcados **Pendiente (real)** hasta que se rediseñen contra el
> esquema correcto — decisión explícita de Luis (2026-09-06): pausar del
> todo por ahora, no re-mapear todavía. Ver `docs/memory.md` para el
> detalle completo del hallazgo.

---

## 1. Mapa General de Cobertura de Tablas (50 Tablas Legacy)

A continuación se detalla la matriz de mapeo que asocia cada una de las 50 tablas históricas con su estado actual de migración, su modelo de datos en Laravel y el proceso que le da soporte:

| # | Tabla Legacy | Estado | Modelo Laravel / Control | Proceso Asociado |
|---|---|---|---|---|
| **1** | `abonopensiones` | **Consolidado** | `App\Models\AbonoPension` | Registro de cargos mensuales y abonos de pensiones. |
| **1a** | `abonouniformes` | **Consolidado** | `App\Models\AbonoUniforme` | Abonos/pagos recibidos por dotación de uniformes. |
| **2** | `actores` | **Consolidado** | `App\Http\Controllers\Api\ContabilidadController::getActores` | Catálogo de solo lectura (1.552 nombres de terceros) — nunca fue una entidad relacional (ninguna otra tabla tiene FK `idactores`), su único uso real es alimentar el autocompletado de "detalle" en Contabilidad y, desde 2026-09-05, también en Ahorro. Sin CRUD propio a propósito. |
| **3** | `agenda` | **Consolidado** | `App\Models\Agenda` | Programación de citas y control de asistencia médica/psicológica. |
| **4** | `ahorro` | **Consolidado** | `App\Models\Ahorro` | Cuenta de ahorros general e histórica de la fundación. |
| **5** | `asientos` | **Consolidado** | `App\Models\Asiento` | Libro diario de contabilidad general (Caja/Bancos) — tabla maestra de fechas y conceptos para `roca`, `jorec` y `diezmos`. |
| **6** | `asientosahorro` | **Consolidado** | `App\Models\AsientoAhorro` | Asientos contables del módulo de ahorros. |
| **7** | `asientosex` | **Consolidado** | `App\Models\AsientoEx` | Asientos contables extraordinarios de la cuenta externa Daniel. |
| **8** | `asociacion` | **Consolidado** | `App\Models\Residente::acudientes()` | Tabla pivote que asocia un residente con sus acudientes. |
| **9** | `asteriscos` | **Soporte** | *Query Builder Directo* | Tabla auxiliar legacy. |
| **10** | `ciudades` | **Soporte** | *Query Builder Directo* | Listado estático de ciudades para el formulario de ingreso. |
| **11** | `cobroalmuerzos` | **Consolidado** | `App\Models\CobroAlmuerzo` | Facturación y cobro de almuerzos extra para residentes con abono contable integrado. |
| **12** | `cobrospension` | **Consolidado** | `App\Models\CobroPension` | Configuración de cobros (monto y día de pago) por residente. |
| **12a** | `cobroslavada` | **Consolidado** | `App\Models\CobrosLavada` | Cargos y abonos para el servicio de lavandería. |
| **13** | `colombia` | **Consolidado** | `App\Models\Colombia` | Movimientos bancarios de la cuenta Bancolombia. |
| **14** | `colpatria` | **Consolidado** | `App\Models\Colpatria` | Movimientos bancarios de la cuenta Colpatria. |
| **15** | `conceptos` | **Consolidado** | `App\Models\Concepto` | Catálogo de conceptos contables usados en asientos. CRUD completo con protección histórica. |
| **16** | `departamentos` | **Soporte** | *Query Builder Directo* | Listado estático de departamentos geográficos. |
| **17** | `detallepedido` | **Pendiente (real)** | `App\Models\DetallePedido` | Detalle de compras a proveedores — desconectado de la operación real, ver aviso arriba. La tabla vigente vive en `u727327027_tienda`. |
| **18** | `detalleventa` | **Pendiente (real)** | `DB::table('detalleventa')` | Detalle de ítems vendidos en el POS — sin actividad desde antes de 2020, ver aviso arriba. La operación real usa `facturas`/`detallefactura` en `u727327027_tienda`. |
| **19** | `diezmos` | **Consolidado** | `App\Models\Diezmo` | Libro contable de diezmos: ingresos/egresos del fondo espiritual con recálculo en cascada. |
| **20** | `efectivo` | **Consolidado (histórico)** | *Sin modelo — no se expone por API* | Libro de caja con 13.750 filas pero **sin movimientos desde 2020-05-22**; el propio legado (`fundacion/tienda/efectivo.php`) ya consulta `efectivorobert`, no esta tabla. Decisión 2026-09-05: dato histórico muerto, no se migra ni se construye pantalla nueva. |
| **21** | `efectivorobert` | **Consolidado** | `App\Models\EfectivoRobert` | Registro de caja menor/efectivo administrado por Robert. |
| **22** | `externa` | **Consolidado** | `App\Models\Externa` | Libro contable para fondos externos o extraordinarios. |
| **23** | `familias` | **Soporte** | *Query Builder Directo* | Familiares y contactos de emergencia del residente. |
| **24** | `historiaclinica` | **Consolidado** | *Conectado a Seguimientos* | Historial clínico general del paciente (Psicología). |
| **25** | `historiaclinicaamp` | **Consolidado** | *Conectado a Seguimientos* | Ampliación de historia clínica psicológica. |
| **26** | `historiaclinicap` | **Consolidado** | *Conectado a Seguimientos* | Fichas específicas de psicología. |
| **27** | `historial` | **Consolidado** | `App\Models\Historial` | Datos de ingreso del residente (drogas, motivos, legal). |
| **28** | `historiali` | **Consolidado** | `App\Models\HistorialIngreso` | Tiempos de estancia (ingresos, reingresos y retiros). |
| **29** | `historialm` | **Consolidado** | `App\Models\HistorialMedico` | Registro de salud, EPS, alergias y vacunas en la admisión. |
| **30** | `historialp` | **Consolidado** | `App\Models\Seguimiento` | Notas históricas de evolución psicológica. |
| **31** | `jorec` | **Consolidado** | `App\Models\Jorec` | Caja contable de la sede JOREC; usada en el cálculo y abono de diezmos vía `idasientos`. |
| **32** | `minutas` | **Consolidado** | `App\Models\Minuta` | Libro de visitas en portería (Minuta de Visitantes). |
| **33** | `pagoproveedores` | **Pendiente (real)** | `App\Models\PagoProveedor` | Pagos a proveedores — desconectado de la operación real, ver aviso arriba. La operación vigente usa `pagos`/`cobros` en `u727327027_tienda`. |
| **34** | `pagos` | **Consolidado** | `App\Http\Controllers\Api\PagoController` | Historial general de cobros y abonos de pensiones. |
| **35** | `pedidos` | **Pendiente (real)** | `App\Models\Pedido` | Compras de abastecimiento — desconectado de la operación real, ver aviso arriba. `u727327027_tienda.pedidos` tiene 3.175 filas vigentes vs. 540 acá. |
| **36** | `permisos` | **Consolidado** | `App\Models\Permiso` | Control de salidas: historial de permisos de residentes con estado Fuera/Retornado, stats y filtros. |
| **37** | `practicantes` | **Consolidado** | `App\Models\Practicante` | Registro y gestión de practicantes y pasantes de psicología con toggle de estado activo/inactivo. |
| **38** | `productos` | **Pendiente (real)** | `App\Models\Producto` | Catálogo de productos — desconectado de la operación real, ver aviso arriba. `u727327027_tienda.productos` (225 filas, esquema distinto) es el vigente. |
| **39** | `proveedores` | **Pendiente (real)** | `App\Models\Proveedor` | Directorio de proveedores — desconectado de la operación real, ver aviso arriba. |
| **40** | `residentes` | **Consolidado** | `App\Models\Residente` | Ficha maestra de identidad del residente. |
| **41** | `roca` | **Consolidado** | `App\Models\Roca` | Caja contable de la sede Jesús es mi Roca; usada en el cálculo y abono de diezmos vía `idasientos`. |
| **42** | `roles` | **Consolidado** | `App\Models\User::getRoleName` | Mapeo de roles (SADMIN, PLANTA, PSICO, CAJERO). |
| **43** | `seguimientos` | **Consolidado** | `App\Models\Seguimiento` | Diario de evolución clínica individual. |
| **44** | `terapiac` | **Consolidado** | `App\Models\TerapiaC` | Fichas de terapias cognitivo-conductuales legacy. |
| **45** | `terapiae` | **Consolidado** | `App\Models\TerapiaE` | Fichas de terapias espirituales/consejeros legacy. |
| **46** | `tienda` | **Pendiente (real)** | `App\Http\Controllers\Api\TiendaController` | Balance de recargas/cargos del POS — **esta tabla en `u727327027_fjemr` está muerta desde antes de 2020**. La `tienda` vigente (86.934 filas, actividad hasta 2026-09-06) vive en `u727327027_tienda`. Ver aviso arriba. |
| **47** | `tipologia` | **Consolidado** | `DB::table('tipologia')` | Solo lectura: Entrada (1) / Salida (2). Expuesta via `/api/tipologias`. |
| **48** | `uniformes` | **Consolidado** | `App\Models\Uniforme` | Inventario, entrega y cobro de uniformes a residentes. |
| **49** | `usuarios` | **Consolidado** | `App\Models\User` | Credenciales de login administrativo. |
| **50** | `validacion` | **Consolidado** | `App\Models\Validacion` (nunca invocado) | Repositorio del hash MD5 de login legacy únicamente (`usuarios.password` viene vacío en el dump — el MD5 real vivía acá, un registro por `idusuarios`). `AuthController::login` la usa directo por `DB::table`, como fallback cuando Bcrypt no matchea, con upgrade silencioso a Bcrypt en `usuarios.password` al validar por esta vía (verificado 2026-09-05). **Corrección 2026-09-06**: la descripción anterior decía "doble función, también auditoría de tokens biométricos" — confirmado que eso es incorrecto, el modelo `Validacion` no se usa en ningún controller y el legado tampoco usa esta tabla para biometría. La biometría real (firma/huella) vive en `residentes.firma_path`/`huella_path`, sin relación con esta tabla — ver módulo 2 más abajo. |

---

## 2. Procesos Consolidados (Backend Laravel 11 + Frontend Angular 19)

Estos procesos ya han sido migrados con éxito, están validados y en estado operativo estable:

### 1. Control de Admisiones (Ingreso Multi-tabla y Biometría)
*   **Lógica Backend:** [IngresoController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/IngresoController.php).
*   **Tablas Afectadas:** `residentes`, `historial`, `historiali`, `historialm`, `asociacion` (crea el residente, el acudiente/usuario, la ficha médica, los motivos de ingreso y la relación familiar de forma atómica bajo una transacción SQL).
*   **Biometría:** Captura de firmas en tiempo real en canvas HTML5 y carga de huellas dactilares, guardados en la tabla `validacion`.
*   **Formatos Legales:** Generación en alta fidelidad con DomPDF del contrato de ingreso legal con firmas biométricas incrustadas ([FormatosController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/FormatosController.php)).

### 2. Gestión Financiera (Pensiones y Abonos)
*   **Lógica Backend:** [PagoController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/PagoController.php).
*   **Tablas Afectadas:** `cobrospension`, `abonopensiones`.
*   **Lógica de Negocio:** Permite configurar a cada residente su valor de pensión y día de cobro mensual. La tarea programada `pensions:charge` ([ChargePensions.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Console/Commands/ChargePensions.php)) genera el cobro de manera automática a los residentes activos (`A`) o especiales (`E`) el día que corresponda. Los abonos y cargos manuales se reflejan de inmediato en su balance en Angular.

### 3. Timeline Clínico (Psicología y Seguimiento)
*   **Lógica Backend:** [SeguimientoController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/SeguimientoController.php).
*   **Tablas Afectadas:** `seguimientos`, `historialp`.
*   **Lógica de Negocio:** Timeline cronológico interactivo donde el psicólogo puede registrar sus sesiones de atención con resumen, evaluación, técnicas y tareas. Permite la exportación en PDF de cada seguimiento clínico individual firmado.

### 4. Punto de Venta (POS Tienda) — ⚠️ PENDIENTE REAL, ver aviso al inicio del documento
*   **Lógica Backend:** [TiendaController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/TiendaController.php).
*   **Tablas Afectadas:** `tienda`, `venta`, `detalleventa`, `productos` — **todas desconectadas de la operación real** (sin actividad desde antes de 2020). Pausado por decisión de Luis (2026-09-06) hasta rediseñar contra `u727327027_tienda` (`facturas`/`detallefactura`/`mayor`).
*   **Lógica de Negocio (como estaba implementada, no vigente):** Los residentes cuentan con una cuenta de tienda (`tienda`) donde se registran recargas (valorentrada) y compras (valorsalida). El POS permite escanear códigos de barras de productos (`Code128`), validar stock contra la base de datos, procesar la transacción atómica y descontar automáticamente del balance del residente.

### 5. Control de Agenda y Citas
*   **Lógica Backend:** [AgendaController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/AgendaController.php).
*   **Tablas Afectadas:** `agenda`.
*   **Lógica de Negocio:** Programación de citas médicas/psicológicas controlando el estado de asistencia (Espera, Atendido, No Asistió).

### 6. Minuta de Portería (Control de Visitantes)
*   **Lógica Backend:** [MinutaController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/MinutaController.php).
*   **Tablas Afectadas:** `minutas`.
*   **Lógica de Negocio:** Registro interactivo de entradas de visitantes asociados a un residente activo de la fundación, con autocompletado predictivo inteligente basado en el histórico de visitantes recurrentes.

### 7. Caja General de Ahorro y Diezmos
*   **Lógica Backend:** [AhorroController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/AhorroController.php).
*   **Tablas Afectadas:** `ahorro`, `asientosahorro`.
*   **Lógica de Negocio:** Control contable unificado de la caja de ahorros general. Permite registrar transacciones de entrada (depósitos) y salida (retiros) bajo transacciones SQL atómicas para evitar inconsistencias en el saldo total recalculado (`acumulado`). Calcula de manera automática un 10% de diezmo espiritual contable para todas las transacciones de entrada, manteniendo el histórico de 8 años de compatibilidad de datos contables.

### 8. Sistema Contable Diario de Cajas y Bancos (ERP Financiero)
*   **Lógica Backend:** [ContabilidadController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/ContabilidadController.php).
*   **Lógica Frontend:** [Contabilidad Component](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/frontend/src/app/components/contabilidad/contabilidad.ts).
*   **Tablas Afectadas:** `colombia`, `colpatria`, `efectivorobert`, `externa`, `asientosex`.
*   **Lógica de Negocio:** Consolida en tiempo real los saldos unificados de los 4 canales financieros principales de la fundación (Bancolombia, Colpatria, Caja General en Efectivo administrada por Robert y la cuenta externa asociada a Daniel). Implementa un algoritmo atómico de **recálculo cronológico en cascada** para transacciones y ediciones históricas, garantizando la exactitud matemática de los libros contables sin impactar en el rendimiento de la base de datos histórica.

### 9. Abastecimiento de Inventario (Compras y Proveedores) — ⚠️ PENDIENTE REAL, ver aviso al inicio del documento
*   **Lógica Backend:** [ProveedorController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/ProveedorController.php) y [PedidoController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/PedidoController.php).
*   **Lógica Frontend:** [Compras Component](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/frontend/src/app/components/compras/compras.ts) y [ComprasDetalle Component](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/frontend/src/app/components/compras/compras-detalle.ts).
*   **Tablas Afectadas:** `pedidos`, `detallepedido`, `proveedores`, `pagoproveedores`, `productos` — **desconectadas de la operación real** (la vigente vive en `u727327027_tienda`: `pedidos` 3.175 filas, `detallepedido` 10.571, `productos` 225, más `facturas`/`detallefactura`/`mayor` sin equivalente acá). `colombia`, `colpatria`, `efectivorobert`, `pagos`, `asientos` (de `u727327027_fjemr`) sí siguen vigentes para el resto de la contabilidad, no forman parte de este hallazgo. Pausado por decisión de Luis (2026-09-06).
*   **Lógica de Negocio (como estaba implementada, no vigente):** Gestión del flujo de compras e ingreso de mercancías. Permite registrar facturas/remisiones de proveedores en estado borrador, buscar productos del catálogo e incorporar unidades de stock en tiempo real mediante sumas dinámicas en el POS de la Tienda. Al finalizar y pagar la factura, de manera atómica se calcula el costo total, se registra el abono del proveedor, se asienta en el egreso de la cuenta bancaria elegida y se ejecuta el recálculo cronológico de saldos en cascada de forma automática.

### 10. Control de Uniformes y Lavandería
*   **Lógica Backend:** [UniformeController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/UniformeController.php) y [LavadaController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/LavadaController.php).
*   **Lógica Frontend:** [Uniformes Component](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/frontend/src/app/components/uniformes/uniformes.ts).
*   **Tablas Afectadas:** `uniformes`, `abonouniformes`, `cobroslavada`, `colombia`, `colpatria`, `efectivorobert`, `asientos`.
*   **Lógica de Negocio:** Consolida en un solo panel de dos pestañas: (1) La entrega física de uniformes dotacionales de 4 piezas (`nuevo`, `antiguo`, `visita`, `buzo`) y el cobro correspondiente con abonos integrados atómicamente a los canales financieros (Bancolombia, Colpatria, Efectivo) mediante recálculos en cascada cronológicos. (2) El registro de cargos por lavandería a residentes asignándolos a fundaciones asociadas (por ejemplo, JOREC) y facilitando sus abonos financieros instantáneos.

### 11. Registro de Diezmos y Ofrendas
*   **Lógica Backend:** [DiezmoController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/DiezmoController.php).
*   **Lógica Frontend:** [Diezmos Component](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/frontend/src/app/components/diezmos/diezmos.ts).
*   **Tablas Afectadas:** `diezmos`, `roca`, `jorec`, `asientos`, `colombia`, `colpatria`, `efectivorobert`.
*   **Modelos Eloquent:** `Asiento`, `Roca`, `Jorec`, `Diezmo`.
*   **Lógica de Negocio:** Permite calcular el 10% sugerido de diezmo espiritual consolidando los ingresos (`valorentrada`) de ambas sedes físicas (`roca` y `jorec`) en un rango de fechas arbitrario. Dado que estas tablas no tienen fecha inline, el join se realiza via `asientos.fecha`. Resta los abonos ya contabilizados en la tabla `diezmos` para presentar el **saldo neto pendiente**. Al registrar un abono, se crea atómicamente un asiento contable en `asientos` y se escribe la salida del canal de origen elegido (Caja Roca, Caja Jorec, Bancolombia, Colpatria o Efectivo Robert) y la entrada en el libro de diezmos, ejecutando el recálculo cronológico en cascada en ambos libros.

### 12. Facturación de Almuerzos Extra
*   **Lógica Backend:** [AlmuerzoController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/AlmuerzoController.php).
*   **Lógica Frontend:** [Almuerzos Component](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/frontend/src/app/components/almuerzos/almuerzos.ts).
*   **Tablas Afectadas:** `cobroalmuerzos`, `colombia`, `colpatria`, `efectivorobert`, `asientos`.
*   **Modelo Eloquent:** `CobroAlmuerzo`.
*   **Lógica de Negocio:** Permite facturar almuerzos extra a residentes activos/especiales (`A`/`E`). Almacena el valor inicial y calcula el saldo pendiente. Los abonos se integran atómicamente con el canal contable elegido (Bancolombia, Colpatria, Efectivo Robert) creando el asiento en `asientos` y ejecutando el recálculo cronológico en cascada.

### 13. Centro de Reportes Consolidado
*   **Lógica Backend:** [ReporteController.php](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/backend/app/Http/Controllers/Api/ReporteController.php).
*   **Lógica Frontend:** [Reportes Component](file:///Users/lgarcia/Documents/GitHub/Softclass/FundacionRebuild/frontend/src/app/components/reportes/reportes.ts).
*   **Tablas Afectadas:** `residentes`, `historiali`, `seguimientos`, `permisos`.
*   **Lógica de Negocio:** Panel unificado de control y analítica de datos en tiempo real:
    *   **Resumen Global:** Tarjetas interactivas de KPIs que calculan residentes activos (incluyendo estados `A` y `E`), inactivos, sesiones de psicología, reingresos y salidas registradas del periodo.
    *   **Historial de Reingresos:** Tabla cronológica compacta con paginación configurable y filtro dinámico. El motivo de retiro se muestra resumido y al hacer clic abre un modal *glassmorphic* con todos los detalles.
    *   **Seguimientos de Psicología:** Listado limpio sin columnas desbordantes, con un botón de detalle que despliega un modal interactivo con el resumen del caso, diagnóstico, técnicas empleadas y tareas asignadas.
    *   **Control de Salidas (Permisos):** Analítica de salidas distinguiendo residentes en estado "Fuera" y "Retornado".
    *   **Estética Premium Multitema:** Soporte de contraste de alta fidelidad en Modo Claro y Oscuro para todos los indicadores y textos del centro analítico.
    *   **Gestión de Estado Manual (Angular Quirks):** Se debe invocar explícitamente `ChangeDetectorRef.detectChanges()` dentro de todos los bloques asíncronas de suscripción (HTTP RxJS) y métodos de interacción (abrir/cerrar modales, paginación, filtros). Esto fuerza la sincronización inmediata del DOM (Zone.js) y previene estados indefinidos de carga (Loading eterno) o desfases en la interfaz.
    *   **Paginación Reactiva Mandatoria:** Todas las tablas de datos (Reingresos, Psicología, Salidas) implementan controles de paginación locales e independientes con selectores de tamaño de página (`pageSize` de 5, 10, 20 o 50 registros) y navegación predictiva, controlados por métodos dinámicos de segmentación (`getPaginated...()`). Esto es indispensable para evitar el desborde vertical de la pantalla y optimizar el rendimiento de renderizado en el navegador.

---

## 3. Procesos Pendientes de Consolidación (Brecha con el Sistema Legacy)

Estos procesos están implementados en el código legacy dentro de la carpeta `fundacion`, pero **aún no se han integrado en la nueva plataforma moderna**:

### 1. Sistema Contable Diario de Cajas y Bancos (ERP Financiero) (Migrado)
*   **Consolidación:** Integrado en Angular y Laravel a través del componente `Contabilidad` y el controlador `ContabilidadController`. Soporta visualización de balances, filtros avanzados de búsqueda y fechas, registro de asientos y edición segura con recálculo en cascada cronológico.

### 2. Ledger de Ahorros de Residentes (Migrado)
*   **Consolidación:** Integrado en Angular y Laravel a través del componente `Ahorro` y el controlador `AhorroController`. Respeta la estructura de balances acumulados y diezmo automático sin alterar la base de datos histórica.

### 3. Minuta de Control de Acceso y Visitantes (Migrado)
*   **Consolidación:** Integrado en Angular y Laravel a través del componente `Minuta` y el controlador `MinutaController`. Permite búsquedas inteligentes y autocompletado en tiempo real.

### 4. Abastecimiento de Inventario (Compras y Proveedores) — ⚠️ PENDIENTE REAL (2026-09-06)
*   **Consolidación:** El módulo `Compras / Stock` está construido e implementado, pero contra la base `u727327027_fjemr` — desconectada de la operación real de compras/proveedores, que vive en `u727327027_tienda` desde antes de 2020. Pausado por decisión de Luis hasta rediseñar contra el esquema correcto. Ver aviso al inicio del documento y `docs/memory.md`/`docs/plan-corte.md` para el detalle completo.

### 5. Control de Uniformes de Residentes (Migrado)
*   **Consolidación:** Completamente migrado mediante el módulo `Dotación / Lavadas`. Soporta la gestión física y de abonos de uniformes.

### 6. Liquidación de Lavandería y Almuerzos Extra (Migrado)
*   **Consolidación:** El servicio de Lavandería (Lavadas) ha sido completamente integrado en el módulo `Dotación / Lavadas`. Queda pendiente el control específico de almuerzos extra y visitas comedor en la cuenta mensual si el usuario lo requiere.

### 7. Registro de Diezmos y Ofrendas (**Migrado**)
*   **Consolidación:** Completamente integrado en el módulo `✝️ Diezmos / Ofrendas` (ruta `/diezmos`). Implementa el cálculo dinámico del 10% sugerido sobre ingresos unificados de sedes (`roca` + `jorec`) con abono atómico multicanal y recálculo cronológico en cascada. ✅

### 8. Carnets de Residentes (PDF) (**Migrado**)
*   **Consolidación:** Totalmente integrado en el listado del `Dashboard`. Permite descargar dinámicamente un PDF con formato oficial (JOREC o JEMR según sede), incluyendo los datos del residente, su EPS, acudiente y código de barras generador PNG codificado a 128 bits de manera offline. ✅

### 9. Terapias Estructuradas (`terapiac`, `terapiae`) (**Migrado**)
*   **Consolidación:** Completamente migrado mediante el módulo `Terapias Estructuradas` (ruta `/terapias`). Soporta el registro de sesiones cognitivo-conductuales (colíder, fallas, observaciones, ayudas) y de acompañamiento espiritual/consejería (resumen, evaluación, técnicas, tareas asignadas) con interfaces de búsqueda y edición. ✅

---

## 4. Próximos Pasos Recomendados

Para continuar la consolidación manteniendo la estabilidad y alta fidelidad del sistema:

1.  **Cobro de Almuerzos Extra (`cobroalmuerzos`):** ✅ Migrado
2.  **Gestión de Practicantes (`practicantes`):** ✅ Migrado
3.  **Conceptos y Tipologías Contables (`conceptos`, `tipologia`):** ✅ Migrado
4.  **Control de Salidas y Permisos (`permisos`):** ✅ Migrado
5.  **Reportes Avanzados (reingresos, psicología, salidas, resumen global):** ✅ Migrado
6.  **Terapias Estructuradas (`terapiac`, `terapiae`):** ✅ Migrado
7.  **Carnets de Residentes (PDF):** ✅ Migrado
8.  **Tienda POS y Compras/Proveedores:** ⚠️ **Pendiente real** — rediseñar contra `u727327027_tienda` (`facturas`/`detallefactura`/`mayor`/`pedidos`/`productos`), la base donde vive la operación vigente. Pausado por decisión de Luis (2026-09-06), ver aviso al inicio del documento.

