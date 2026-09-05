# Memoria del Proyecto - Fundación Rebuild

## Estado Actual
Migración completada de los módulos críticos del sistema PHP legado a una arquitectura moderna (Laravel 11 + Angular 19). El sistema es ahora 100% funcional para la operación diaria, con alta fidelidad visual y seguridad mejorada.

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
