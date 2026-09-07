# Tickets: Consolidación de recálculo en cascada (libros contables)

**Spec**: `docs/specs/consolidacion-cascada-contable.md` (Approved)
**Fecha de apertura**: 2026-09-07
**PM**: desglose de las 6 fases del diseño del Arquitecto en tareas
ejecutables. 6 tareas — dentro del límite de sub-batch (≤6), no hace falta
partir en lotes más chicos.

**Orden de ejecución**: T1 es bloqueante para T2-T6. T2, T3 y T4 pueden
paralelizarse entre Backend Devs distintos una vez que T1 está en `dev`
(no comparten archivo de controller). T5 depende de T3 y T4 (necesita que
`ContabilidadController`/`DiezmoController` ya usen el servicio y sus
configs de canal estén consolidadas antes de migrar Roca/Jorec/Diezmo).
T6 depende de T3 (reutiliza los configs de Colombia/Colpatria/Efectivo que
T3 deja en `LedgerChannel`).

---

## T1 — Construir el servicio base y sus tests unitarios
```
Task: Crear CascadeLedgerService + LedgerChannelConfig + LedgerChannel con
      tests unitarios aislados
Agent: Backend Dev
Depends on: none
Acceptance:
  - LedgerChannelConfig existe como value object inmutable (constructor
    promotion, readonly) con los campos: table, idColumn, dateColumn,
    entradaColumn, salidaColumn, acumuladoColumn, joinTable,
    joinLocalKey, joinForeignKey.
  - CascadeLedgerService::recalculateFrom(config, fecha, id) implementa el
    algoritmo ya validado (ubicar `prev` por fecha desc + id desc con
    desempate correcto, recalcular en cascada solo las filas
    posteriores), soportando tanto tabla simple (sin join) como tabla con
    join (joinTable presente).
  - LedgerChannel::config(nombre) devuelve el LedgerChannelConfig correcto
    para cada uno de los 8 canales: colombia, colpatria, efectivo,
    externa, ahorro, roca, jorec, diezmo. Nombre de canal desconocido
    lanza una excepción clara (no un null silencioso).
  - Tests unitarios (sin HTTP, sin controller) cubren: entrada
    retroactiva antes de la primera fila, entrada retroactiva entre dos
    filas existentes, empate de fecha con desempate por id, primera fila
    de una tabla vacía (sin `prev`, acumulado base = 0), variante con
    join (usando el config de `externa` o `ahorro` como caso real).
  - Ningún controller existente se modifica en este ticket — el servicio
    vive en paralelo al código viejo hasta T2-T6.
Files:
  - backend/app/Services/Ledger/CascadeLedgerService.php (nuevo)
  - backend/app/Services/Ledger/LedgerChannelConfig.php (nuevo)
  - backend/app/Services/Ledger/LedgerChannel.php (nuevo)
  - backend/tests/Unit/Services/Ledger/CascadeLedgerServiceTest.php (nuevo)
```

## T2 — Migrar AhorroController
```
Task: Reemplazar la implementación de cascada de AhorroController por
      CascadeLedgerService
Agent: Backend Dev
Depends on: T1
Acceptance:
  - AhorroController::store() (y update()/donde aplique) llama a
    CascadeLedgerService::recalculateFrom() con LedgerChannel::config('ahorro')
    en vez de construir la query de cascada a mano.
  - Cero lógica de cascade recalculation propia queda en AhorroController.
  - AhorroTest.php pasa sin modificar su intención — incluyendo
    test_acumulado_stays_chronologically_consistent_with_a_backdated_entry
    (el regression test del bug real de esta semana), que sigue en verde
    contra la implementación nueva.
  - Suite completa de backend (154+ tests) pasa.
Files:
  - backend/app/Http/Controllers/Api/AhorroController.php
```

## T3 — Migrar ContabilidadController (Colombia/Colpatria/Efectivo/Externa)
```
Task: Reemplazar los 4 métodos privados de cascada de ContabilidadController
      por CascadeLedgerService
Agent: Backend Dev
Depends on: T1
Acceptance:
  - store() y update() para los 4 canales (colombia, colpatria, efectivo,
    externa) llaman a CascadeLedgerService::recalculateFrom() con el
    LedgerChannel::config() correspondiente.
  - Los 4 métodos privados cascadeRecalculate{Colombia,Colpatria,Efectivo,Externa}
    quedan eliminados de ContabilidadController.
  - Se agrega al menos un test de regresión por canal (4 en total)
    replicando el caso de entrada retroactiva, si no existe ya cobertura
    equivalente.
  - Suite completa de backend pasa.
Files:
  - backend/app/Http/Controllers/Api/ContabilidadController.php
  - backend/tests/Feature/ContabilidadTest.php (tests nuevos si no existen)
```

## T4 — Eliminar la duplicación en DiezmoController (Colombia/Colpatria/Efectivo)
```
Task: Reemplazar los 3 métodos duplicados de cascada en DiezmoController
      por CascadeLedgerService, usando el MISMO LedgerChannel::config() que T3
Agent: Backend Dev
Depends on: T1 (puede correr en paralelo a T3, pero debe fusionarse después
      de T3 para confirmar que ambos controllers apuntan al mismo config —
      coordinar el merge con quien tome T3 si son personas distintas)
Acceptance:
  - Los 3 métodos privados cascadeRecalculate{Colombia,Colpatria,Efectivo}
    quedan eliminados de DiezmoController — cero copia propia de esta
    lógica en el archivo.
  - Las llamadas usan exactamente el mismo LedgerChannel::config('colombia'/
    'colpatria'/'efectivo') que ContabilidadController (T3) — este es el
    punto central del ticket: verificar que no queden dos configs
    distintos para el mismo canal.
  - Suite completa de backend pasa, incluyendo cualquier test existente de
    DiezmoController que ejercite estos 3 canales.
Files:
  - backend/app/Http/Controllers/Api/DiezmoController.php
```

## T5 — Migrar Roca/Jorec/Diezmo de recómputo completo a cascada parcial
```
Task: Reemplazar el algoritmo de recómputo completo (Roca/Jorec/Diezmo) por
      CascadeLedgerService, con verificación de equivalencia previa
Agent: Backend Dev — ticket de mayor cuidado, no tratar como mecánico
Depends on: T3, T4 (necesita el patrón y los configs ya consolidados)
Acceptance:
  - ANTES de tocar código: sobre el dataset real del MySQL local de
    pruebas, correr el algoritmo viejo (recómputo completo) y el nuevo
    (CascadeLedgerService::recalculateFrom) sobre las mismas tablas
    roca/jorec/diezmos y comparar el acumulado resultante fila por fila.
    Documentar el resultado de esta comparación en el PR/commit — si hay
    una sola diferencia, el ticket se bloquea y se escala al Arquitecto
    antes de continuar, no se "ajusta a mano" para que cierre.
  - Solo si la comparación cierra en cero diferencias: reemplazar
    cascadeRecalculate{Roca,Jorec,Diezmo}() por la llamada al servicio.
  - Se agrega un test de regresión por canal (backdated entry), igual que
    los demás canales — hoy no existe ninguno porque el algoritmo viejo no
    tenía ese caso límite (recorría todo desde cero siempre).
  - Suite completa de backend pasa.
Files:
  - backend/app/Http/Controllers/Api/DiezmoController.php
  - backend/tests/Feature/DiezmoTest.php (tests nuevos)
```

## T6 — Migrar las llamadas embebidas (Almuerzo/Uniforme/Pedido)
```
Task: Reemplazar la construcción manual de $channelTable/$primaryKey y la
      cascada inline en Almuerzo/Uniforme/Pedido por LedgerChannel + CascadeLedgerService
Agent: Backend Dev
Depends on: T3 (reutiliza los configs de colombia/colpatria/efectivo)
Acceptance:
  - AlmuerzoController, UniformeController y PedidoController resuelven el
    canal elegido vía LedgerChannel::config($channel) — con el mismo mapeo
    $channel === 'efectivo' ? 'efectivorobert' : $channel que ya usan hoy,
    para no romper el contrato con Angular (el value 'efectivo' que manda
    el frontend no cambia).
  - Cero construcción manual de $channelTable/$primaryKey ni cascada
    inline en los 3 archivos.
  - Suite completa de backend pasa (AlmuerzoTest.php, UniformeTest.php, y
    el test de Pedido/Compras correspondiente).
Files:
  - backend/app/Http/Controllers/Api/AlmuerzoController.php
  - backend/app/Http/Controllers/Api/UniformeController.php
  - backend/app/Http/Controllers/Api/PedidoController.php
```

## Tech Writer (en paralelo, arranca junto con T1)
```
Task: Documentar CascadeLedgerService en docs/architecture.md
Agent: Tech Writer
Depends on: T1 (necesita la interfaz final del servicio, puede arrancar el
      borrador con el diseño de la spec mientras T1 se implementa)
Acceptance:
  - docs/architecture.md tiene una sección nueva ("Patrón: recálculo en
    cascada de libros contables" o similar) explicando qué es
    CascadeLedgerService, cuándo usarlo, y remite a
    docs/specs/consolidacion-cascada-contable.md para el detalle completo
    — objetivo: que nadie vuelva a reimplementar este algoritmo a mano en
    un controller futuro.
Files:
  - docs/architecture.md
```

## Estado
| Ticket | Estado |
|---|---|
| T1 | **Hecho** (2026-09-07) — `CascadeLedgerService`/`LedgerChannelConfig`/`LedgerChannel` creados, 6 tests unitarios nuevos + 154 existentes en verde (160/160). Cero controllers tocados. Revisado por Senior Reviewer. |
| T2 | Desbloqueado — listo para asignar |
| T3 | Desbloqueado — listo para asignar |
| T4 | Desbloqueado — listo para asignar (coordinar merge con T3, ver nota del ticket) |
| T5 | Bloqueado por T3, T4 |
| T6 | Bloqueado por T3 |
| Tech Writer | Desbloqueado — puede arrancar |

Al cerrar cada ticket: Senior Reviewer revisa antes de pasar al siguiente
dependiente (no acumular 2-3 tickets sin review y descubrir un defecto
compartido tarde). Reporte consolidado al Arquitecto al cerrar la fase
completa (T1-T6 + doc), no por ticket individual — ver "Notificaciones" en
el `CLAUDE.md` raíz.
