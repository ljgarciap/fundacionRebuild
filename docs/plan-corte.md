# Plan de Corte a Producción — Fundación Rebuild

Estado: **esqueleto, sin completar**. Creado 2026-09-05 al retomar el
proyecto — la auditoría de ese día encontró que no existía ningún plan de
corte, staging ni CI/CD, pese a que la migración de módulos críticos está
casi terminada según `consolidado.md`. Este documento no se puede completar
sin decisiones de negocio de Luis — están marcadas como preguntas abiertas.

## 1. Operación actual del sistema legado (pendiente de confirmar con Luis)
- ¿Dónde corre hoy en producción el sistema PHP monolítico — mismo hosting
  de origen del dump (`u727327027_...`, aparenta ser cPanel/Hostinger) u otro?
- ¿Quién lo administra/tiene acceso? ¿Hay backups automáticos corriendo hoy
  sobre esa base, independientes de este proyecto?
- ¿Cuál es el volumen de escritura diaria real (ingresos, pagos, ventas de
  tienda, seguimientos de psicología) que seguiría entrando al sistema viejo
  mientras se termina de validar el nuevo?

## 2. Estrategia de corte (a decidir)
Opciones típicas para este tipo de refactor — **ninguna elegida todavía**:
- **Big bang**: se congela el sistema viejo un fin de semana, se migra el
  dump final, se activa el nuevo sistema. Simple, pero exige ventana de
  downtime y que todos los módulos estén 100% validados antes.
- **Corte por módulo**: se van apagando módulos del viejo sistema a medida
  que su equivalente en Laravel/Angular pasa QA (ej. primero Tienda, después
  Finanzas). Más lento, pero de menor riesgo — exige que ambos sistemas
  puedan convivir leyendo la misma base sin pisarse mientras dure la transición.
- **Piloto**: una sede (JOREC o Jesús es mi Roca) migra primero, la otra
  sigue en el sistema viejo un tiempo. Requiere separar datos por sede de
  forma limpia en el esquema legado — a confirmar si eso ya es así.

## 3. Validación previa al corte (bloqueante, sin importar la estrategia elegida)
- [ ] Cobertura de tests del proceso de **Ingreso** (multi-tabla, atómico,
      con biometría) — hoy no tiene test dedicado, es el proceso más crítico.
- [ ] Resolver `actores` y `efectivo` (ver `consolidado.md` — dos tablas
      "Pendiente" sin decisión de si siguen vigentes en la operación real).
- [ ] Confirmar mecanismo real de migración de contraseñas MD5 → Bcrypt
      (ver `CLAUDE.md` del proyecto, sección de reglas específicas).
- [ ] Ambiente de staging con datos reales (o una copia fiel) antes del corte
      — hoy todo el trabajo es contra MySQL local con el dump de referencia.
- [ ] Ronda de QA formal por módulo contra los criterios de aceptación de
      cada proceso migrado (no solo verificación técnica de que compila/corre).

## 4. Plan de rollback (a definir)
- ¿Cuánto tiempo se mantiene el sistema viejo disponible en modo
  solo-lectura después del corte, por si aparece un caso no contemplado?
- ¿Quién tiene la decisión de activar un rollback si algo falla el día del
  corte, y con qué criterio?

---
**Próximo paso**: agendar un daily con Luis específicamente para esta
sección — no es una decisión técnica que el Arquitecto pueda resolver solo
(ver "Autonomía del equipo de agentes" en el `CLAUDE.md` del workspace,
categoría "decisión de negocio").
