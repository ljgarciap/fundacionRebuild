# Spec: Consolidación de recálculo en cascada (libros contables)

**Date**: 2026-09-07
**Requested by**: Luis
**Status**: Approved — diseño técnico adjunto, pendiente de desglose PM
**Project**: Fundación Rebuild

## Contexto — por qué esta spec y no una migración de esquema

Luis planteó si conviene diseñar un esquema nuevo y migrar la base legada en
vez de seguir adaptando el sistema nuevo al esquema original. Análisis
conjunto (2026-09-06/07): de los bugs reales encontrados en el QA/comparación
en vivo de esta semana, **ninguno lo hubiera evitado un esquema nuevo** —
son migraciones olvidadas (`tipo_sanguineo`, `encargado`) o lógica de
aplicación inconsistente (Ahorro). El único hallazgo que sí apunta a un
problema real de diseño es la duplicación del algoritmo de recálculo en
cascada entre controllers — y **eso se resuelve sin tocar el esquema legado**,
consolidando la lógica en el código Laravel. Esta spec es esa consolidación:
la alternativa de bajo riesgo a la migración completa de esquema, que **queda
descartada** para el resto del sistema (ver decisión en `docs/memory.md`
2026-09-07). Tienda/Compras se queda fuera de esta spec y de cualquier
refactor por ahora — sistema aparte, obsoleto, sin actividad desde antes de
2020, pausado por decisión explícita de Luis (2026-09-06).

## Problem
El algoritmo "ubicar el acumulado previo por fecha (desc) + id (desc), y
recalcular en cascada todas las filas posteriores" está **reimplementado de
forma independiente al menos 13 veces** en 6 controllers distintos, en vez de
vivir en un solo lugar:

| Controller | Implementaciones | Canal/tabla |
|---|---|---|
| `ContabilidadController.php` | 4 métodos privados (`cascadeRecalculateColombia`, `cascadeRecalculateColpatria`, `cascadeRecalculateEfectivo`, `cascadeRecalculateExterna`) | `colombia`, `colpatria`, `efectivorobert`, `externa` |
| `DiezmoController.php` | 6 métodos privados (`cascadeRecalculateRoca`, `cascadeRecalculateJorec`, `cascadeRecalculateDiezmo`, **+ `cascadeRecalculateColombia`, `cascadeRecalculateColpatria`, `cascadeRecalculateEfectivo` duplicados del punto anterior**) | `roca`, `jorec`, `diezmos`, y de nuevo `colombia`/`colpatria`/`efectivorobert` |
| `AlmuerzoController.php` | 1 implementación inline (`store()`) | `cobroalmuerzos` |
| `UniformeController.php` | 1 implementación inline (`store()`) | `abonouniformes` |
| `PedidoController.php` | 1 implementación inline (`store()`) | `pedidos`/canal de pago |
| `AhorroController.php` | 1 implementación (corregida 2026-09-06 tras bug real) | `ahorro`/`asientosahorro` |

Los canales `colombia`, `colpatria` y `efectivorobert` tienen **su lógica de
cascada duplicada palabra por palabra entre dos archivos distintos**
(`ContabilidadController` y `DiezmoController`) — si se corrige un bug en
uno, nada obliga a corregirlo en el otro. Esto es exactamente lo que pasó con
`AhorroController`: el resto de los libros ya usaban el patrón correcto
(ordenar por fecha, no por id de inserción), Ahorro quedó atrás y produjo
acumulados incorrectos con una entrada con fecha retroactiva (bug real,
corregido 2026-09-06, ver `docs/memory.md`). No hay ninguna garantía de que
no exista ya, o no vaya a existir, el mismo desvío en cualquiera de las otras
12 implementaciones.

## Solution summary
Extraer el algoritmo a un único servicio compartido de Laravel (capa de
aplicación, **sin tocar el esquema legado** — cumple la regla dura del
proyecto de nunca alterar tablas existentes), parametrizado por: tabla,
join opcional (ej. `asientosahorro`, `asientosex`), columna de fecha, columna
de id, columnas de entrada/salida, columna de acumulado. Cada controller
pasa a llamar a ese servicio en vez de mantener su propia copia del
algoritmo. Un solo lugar, una sola prueba de regresión por tipo de bug, un
solo fix futuro que cubre los 6 canales a la vez.

## Users and roles
Sin cambio visible para el usuario final ni para los roles (Finanzas,
Contabilidad, Cajero). Es un refactor interno — beneficia indirectamente a
todos los roles que operan Contabilidad, Diezmos, Almuerzos, Uniformes,
Pedidos y Ahorros, al eliminar una clase entera de bug (acumulados
inconsistentes) en todos esos módulos a la vez.

## Acceptance criteria
- [ ] Existe un único punto de implementación del algoritmo de cascada
      (servicio o trait, a decidir por el Arquitecto — ver Open questions).
- [ ] Los 6 controllers (`Contabilidad`, `Diezmo`, `Almuerzo`, `Uniforme`,
      `Pedido`, `Ahorro`) usan esa implementación única; cero copias propias
      del algoritmo restantes en ningún controller.
- [ ] Las 3 duplicaciones exactas entre `ContabilidadController` y
      `DiezmoController` (Colombia/Colpatria/Efectivo) quedan reducidas a una
      sola llamada compartida desde ambos.
- [ ] Los 154+ tests de backend existentes siguen pasando sin modificar su
      intención (solo refactor de implementación, no de contrato).
- [ ] Se agrega al menos un test de regresión por canal (6 en total,
      mínimo) replicando el caso que rompió a Ahorro: insertar una entrada
      con fecha retroactiva y verificar que el acumulado de las filas
      posteriores se recalcula correctamente.
- [ ] El esquema de base de datos legado permanece sin ningún cambio
      (columnas, tablas, tipos) — este refactor es 100% capa de aplicación.
- [ ] Ningún endpoint cambia de firma/contrato — Angular no requiere ningún
      cambio para consumir el resultado.

## Edge cases and error scenarios
- Entrada con fecha retroactiva (el caso que ya rompió Ahorro) — cubierto
  arriba.
- Dos entradas el mismo día: el desempate debe seguir siendo por id
  (orden de inserción), igual que hoy.
- Edición de una entrada que mueve su fecha hacia atrás o adelante (afecta
  el rango a recalcular, no solo el punto de inserción) — verificar que
  `update()` en cada controller también pase por el servicio compartido,
  no solo `store()`.
- Eliminación de una entrada (si existe la operación en algún canal):
  confirmar si hoy dispara recálculo cronológico o no, y mantener el mismo
  comportamiento tras el refactor (no ampliar alcance sin decisión aparte).
- Canal con cero entradas previas (primera fila): acumulado base = 0, ya
  cubierto por el patrón actual (`$prevTrans ? ... : 0`), debe preservarse.
- Falla de la transacción a mitad de la cascada: debe seguir revirtiendo
  atómicamente (los controllers actuales ya envuelven esto en
  `DB::transaction`, el servicio nuevo debe operar dentro de esa misma
  transacción, no abrir la suya).

## Out of scope
- Tienda/Compras: **fuera de esta spec y de cualquier refactor por ahora**.
  Sistema aparte, esquema propio en `u727327027_tienda`, sin actividad real
  desde antes de 2020, pausado por decisión explícita de Luis (2026-09-06).
  No se toca ni se referencia desde el servicio nuevo.
- Cualquier cambio al esquema de base de datos legado (columnas, tablas,
  tipos, nombres). Esta spec es explícitamente la alternativa a eso.
- Migración completa a un esquema moderno — descartada para el resto del
  sistema tras el análisis 2026-09-06/07 (ver Contexto arriba).
- Cambios de UI/UX en Angular — ningún componente frontend se toca.
- Ampliar el algoritmo a nuevas funcionalidades (ej. deshacer/auditoría de
  cascada) — solo consolidar lo que ya existe.

## Open questions
- [Architect] ¿Servicio inyectable (`App\Services\CascadeLedgerService`) o
  trait (`RecalculatesCascadeBalance`) compartido entre controllers? Un
  servicio permite testear en aislamiento sin instanciar el controller; un
  trait es más liviano pero repite la dependencia en cada clase. Recomiendo
  servicio inyectable — más fácil de mockear en tests unitarios.
- [Backend] Confirmar si `update()`/eliminación existen para los 6 canales
  y si ya disparan recálculo — el hallazgo de duplicación se centró en
  `store()`, falta auditar `update()`/`destroy()` en cada controller antes
  de migrarlos.
- [QA] ¿Se corre el batch completo de comparación en vivo (legado vs.
  nuevo) para estos 6 módulos después del refactor, o alcanza con los tests
  de regresión de backend dado que ya se validaron en vivo esta semana?

## Diseño Técnico (Arquitecto, 2026-09-07)

### Hallazgo adicional que cambia el alcance: no es un algoritmo, son dos
Al leer las 13 implementaciones completas (no solo el patrón general) para
diseñar la interfaz del servicio, aparece una segunda variante que la spec
original no distinguía:

- **Cascada parcial desde el punto de inserción** (`Colombia`, `Colpatria`,
  `EfectivoRobert`, `Externa`, `Ahorro`, y las 3 llamadas embebidas de
  `Almuerzo`/`Uniforme`/`Pedido`, que reutilizan estos mismos 3 canales) —
  ubica el acumulado justo antes de la fecha/id nuevo y recalcula solo las
  filas posteriores. Es el patrón correcto, ya probado, el que corrigió el
  bug de Ahorro.
- **Recómputo completo desde cero** (`Roca`, `Jorec`, `Diezmo`, las 3 en
  `DiezmoController`) — ignora `$fecha`/`$id`, recorre y recalcula **la
  tabla entera** desde `acumulado = 0` en cada escritura, sin excepción.

Ambas convergen al mismo resultado numérico si la tabla está bien ordenada
(recómputo completo es, de hecho, más "a prueba de bugs" por construcción
— no puede fallar el mismo desvío que tuvo Ahorro, porque no depende de
ubicar correctamente un punto de partida). Pero recómputo completo es
**O(tabla completa) en cada escritura**, no O(filas posteriores) — hoy no
importa (tablas de cientos/pocos miles de filas), pero es una segunda
implementación con su propio contrato de rendimiento que el servicio
consolidado tiene que decidir si absorbe o preserva.

**Decisión de diseño**: estandarizar todo sobre el algoritmo de cascada
parcial (ya es el que usan 9 de las 12 llamadas, y es el patrón que ya
demostró ser correcto en producción esta semana). Migrar `Roca`/`Jorec`/
`Diezmo` de recómputo completo a cascada parcial **es parte de esta spec**,
no un extra — de lo contrario quedarían 3 implementaciones fuera del
servicio único y el objetivo ("cero copias propias restantes") no se
cumple. Es un cambio de algoritmo, no solo de ubicación del código, así
que lleva su propia verificación de equivalencia antes de eliminar el
código viejo (ver Fase 3 abajo) — no se asume que da lo mismo, se prueba.

### Interfaz del servicio
Un servicio inyectable (no trait — se resolvió la open question de la spec
a favor de un servicio, testeable en aislamiento sin instanciar ningún
controller) más un value object de configuración por canal:

```php
namespace App\Services\Ledger;

final class LedgerChannelConfig
{
    public function __construct(
        public readonly string $table,            // 'colombia'
        public readonly string $idColumn,          // 'idcolombia'
        public readonly string $dateColumn,        // 'fecha'
        public readonly string $entradaColumn = 'valorentrada',
        public readonly string $salidaColumn  = 'valorsalida',
        public readonly string $acumuladoColumn = 'acumulado',
        // Solo para canales donde la fecha vive en una tabla de asiento
        // join-eada (Externa->asientosex, Ahorro->asientosahorro,
        // Roca/Jorec/Diezmo->asientos):
        public readonly ?string $joinTable = null,
        public readonly ?string $joinLocalKey = null,   // FK en $table
        public readonly ?string $joinForeignKey = null, // PK en $joinTable
    ) {}
}

class CascadeLedgerService
{
    // Único método público de escritura. Reemplaza las 13 implementaciones.
    public function recalculateFrom(LedgerChannelConfig $config, string $fecha, int $id): void;
}

// Registro central — un solo lugar con los 8 canales conocidos, en vez de
// repetir los nombres de columna en cada controller que hoy arma su propio
// $channelTable/$primaryKey a mano (Almuerzo/Uniforme/Pedido).
class LedgerChannel
{
    public static function config(string $name): LedgerChannelConfig; // 'colombia'|'colpatria'|'efectivo'|'externa'|'ahorro'|'roca'|'jorec'|'diezmo'
}
```

`recalculateFrom()` implementa exactamente el algoritmo ya validado
(ubicar `prev` por `fecha desc, id desc` con el desempate `id < $id` /
`id >= $id` según corresponda, recalcular en cascada solo lo posterior) —
funciona igual para tabla simple (`where` directo) y para tabla con join
(`joinTable`), unificando lo que hoy son código duplicado y código
join-eado como dos ramas si-el-config-trae-join dentro de una sola
implementación.

Los controllers dejan de construir `$channelTable`/`$primaryKey` a mano
(patrón hoy repetido en Almuerzo/Uniforme/Pedido) y pasan a resolver
`LedgerChannel::config($channel)` — elimina también esa segunda forma de
duplicación, más chica pero real.

### Fases de migración (cada una termina con suite completa en verde antes de la siguiente)
1. **Construir el servicio** (`CascadeLedgerService`, `LedgerChannelConfig`,
   `LedgerChannel`) con su propia batería de tests unitarios aislados
   (sin controller, sin HTTP) cubriendo: entrada retroactiva, empate de
   fecha (desempate por id), primera fila de la tabla (sin `prev`),
   variante con join. Ningún controller se toca todavía.
2. **Migrar `AhorroController`** — superficie más chica, ya tiene el test
   de regresión del bug real de esta semana como canario. Confirmar que
   ese test sigue en verde llamando al servicio nuevo.
3. **Migrar los 4 métodos de `ContabilidadController`** (Colombia/
   Colpatria/Efectivo/Externa) — la de-duplicación más grande de una sola
   vez.
4. **Migrar los 3 métodos duplicados de `DiezmoController`**
   (Colombia/Colpatria/Efectivo) para que llamen al **mismo**
   `LedgerChannel::config()` que ya usa `ContabilidadController` — este
   paso es el que elimina la duplicación literal entre los dos archivos,
   el hallazgo original que motivó la spec.
5. **Migrar Roca/Jorec/Diezmo de recómputo completo a cascada parcial**:
   antes de borrar el código viejo, correr ambos algoritmos sobre el mismo
   snapshot de datos reales (los que ya están en el MySQL local de
   pruebas) y comparar los acumulados resultantes fila por fila — deben
   ser idénticos. Solo si la comparación cierra en cero diferencias se
   reemplaza el método viejo por la llamada al servicio.
6. **Migrar las 3 llamadas embebidas** (`Almuerzo`, `Uniforme`, `Pedido`)
   para que resuelvan el canal vía `LedgerChannel::config()` y llamen al
   servicio, en vez de construir `$channelTable`/`$primaryKey` a mano.

En cada fase: correr la suite completa (154+ tests) + el test de
regresión del canal correspondiente (criterio de aceptación de la spec)
antes de avanzar a la fase siguiente. Ningún paso toca el esquema de base
de datos.

### Riesgo y reversibilidad
Cada fase es un commit independiente, mecánico y con test en verde antes
de seguir — revertir un paso puntual (`git revert`) no afecta a los
anteriores. No hay migración de datos en ningún punto (0 cambios de
esquema), así que no hay ventana de riesgo sobre datos reales en ningún
momento del proceso — a diferencia de la migración de esquema completa que
se descartó, esto se puede pausar entre fases sin dejar nada a mitad de
camino.

### Para PM — desglose sugerido de tickets
1. `CascadeLedgerService` + `LedgerChannelConfig` + `LedgerChannel` + tests
   unitarios aislados (Fase 1).
2. Migrar `AhorroController` (Fase 2).
3. Migrar `ContabilidadController` (Fase 3).
4. Migrar duplicación en `DiezmoController` — Colombia/Colpatria/Efectivo
   (Fase 4).
5. Migrar Roca/Jorec/Diezmo con verificación de equivalencia (Fase 5) —
   marcar como el ticket de mayor cuidado, no mecánico como los anteriores.
6. Migrar Almuerzo/Uniforme/Pedido (Fase 6).

Los tickets 2-6 son independientes entre sí una vez que el 1 está en
`dev` — se pueden paralelizar entre Backend Devs si hay más de uno
disponible, siempre que cada uno corra la suite completa antes de
integrar (evitar pisarse el mismo `CascadeLedgerService` a medio validar).

## References
- Hallazgo y fix original: `AhorroController.php` (bug de cascada,
  corregido 2026-09-06), `docs/memory.md` (entrada 2026-09-06).
- Duplicación confirmada: `ContabilidadController.php:358-484`,
  `DiezmoController.php:250-380`, `AlmuerzoController.php:~150-195`,
  `UniformeController.php:~170-212`, `PedidoController.php:~260-300`.
- Decisión de descartar migración completa de esquema:
  `docs/memory.md` (entrada 2026-09-06/07), este mismo documento (sección
  Contexto).
- Decisión de pausar Tienda/Compras: `docs/consolidado.md` (aviso al
  inicio del documento), `docs/plan-corte.md`.
