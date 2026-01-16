# docs/03_pipeline_steps.md

# Pasos de implementación (plan ejecutable)

## Objetivo
Construir el pipeline V2 con entregables claros, priorizando:
- gobernanza y trazabilidad
- modelo ER estable
- salida consumible para BI/alertas

---

## Fase 1 — Fundaciones (Semana 1)
### 1.1 Definir estándares
- Naming de tablas y columnas (snake_case)
- Tipos (timestamp, string, int), timezone
- IDs: UUID vs hash determinístico (decisión explícita)

### 1.2 Crear esquema físico (DDL)
Crear tablas:
- Catálogos: `systems`, `subsystems`, `components`, `action_types`
- Activos: `machines`, `machine_systems`
- Transaccional: `records`, `jobs`, `actions`

### 1.3 Cargar master data inicial
- bootstrap de taxonomías
- carga de máquinas
- asociación `machine_systems`

**Entregable**
- Base lista para recibir transacciones
- Validaciones FK activas (si el motor lo soporta)

---

## Fase 2 — Ingesta transaccional (Semana 2)
### 2.1 Ingesta → staging
- Crear staging “raw” por fuente
- Guardar metadatos de ingesta (fecha, run_id, archivo, etc.)

### 2.2 Transformación a RECORDS
- Mapeo de columnas fuente → record
- Normalización de fechas e ids de máquina

**Entregable**
- `records` poblado consistentemente
- Check: `records.machine_id` válido

---

## Fase 3 — Construcción de JOBS (Semana 3)
### 3.1 Estrategia de segmentación
- Si la fuente ya trae “tareas”: map directo.
- Si la fuente es texto libre:
  - reglas de chunking
  - diccionario
  - heurísticas (keywords por subsystem)
  - opcional: LLM para segmentación (sin bloquear el pipeline)

### 3.2 Asignación de scope
- Resolver `system_id` y/o `subsystem_id`
- Si no se puede, marcar como “unknown” (pero nunca dejar el job sin scope)

**Entregable**
- `jobs` poblado
- Gate: scope válido + machine_systems coherente

---

## Fase 4 — Construcción de ACTIONS (Semana 4)
### 4.1 Extracción de acciones
- Map de verbos/acciones → `action_types`
- Target técnico:
  - subsystem obligatorio
  - component opcional

### 4.2 Reglas de consistencia
- action.subsystem consistente con job.scope
- action.component pertenece a action.subsystem

**Entregable**
- `actions` poblado
- Vista `vw_actions_enriched` lista para BI

---

## Fase 5 — Serving + alertas (Semana 5+)
### 5.1 Serving
- `vw_actions_enriched`
- `fact_actions_daily` (agregados)

### 5.2 Alertas
- reglas sobre `fact_actions_daily` o `vw_actions_enriched`
- umbrales por máquina/sistema/subsystem

**Entregable**
- “Alertas como producto” soportadas sin tocar texto libre

---

## Reprocesos y backfills (desde el día 1)
Definir desde el inicio:
- cómo reimportar un período (semanas/meses)
- idempotencia (upserts por claves naturales)
- auditoría de runs (tabla `pipeline_runs` recomendada)

---

## Mínimos técnicos no negociables
- PK/FK definidas y respetadas
- Gates bloqueantes ejecutables
- Logging de pipeline + métricas de calidad (volúmenes, nulos, duplicados)
