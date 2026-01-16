# docs/02_data_flow.md

# Flujo de datos end-to-end (desde ingreso hasta consumo)

## Objetivo
Estandarizar el proceso para que cualquier entrada (Excel/ERP/API) termine en un modelo consistente: **Records → Jobs → Actions**, gobernado por catálogos técnicos.

---

## Vista general (medallion conceptual)
- **Bronze:** evidencia cruda + metadatos de ingesta.
- **Silver:** normalización + preprocesamiento + IDs canónicos.
- **Gold (Core):** tablas del ER (catálogos + transaccional) listas para BI/alertas.

Este documento aterriza el flujo usando las tablas del ER.

---

## Paso 0 — Setup (master data)
**Input**
- Catálogo técnico (systems/subsystems/components)
- Catálogo de tipos de acción (action_types)
- Máquinas (machines)
- Configuración por máquina (machine_systems)

**Output**
- Tablas maestras listas y auditables.

**Resultado operacional**
- Puedes validar cualquier transacción contra la realidad de la máquina.

---

## Paso 1 — Ingesta de eventos (create RECORDS)
**Input**
- Fuente: orden de trabajo, ticket, excel semanal, etc.
- Campos mínimos: identificador de máquina y ventana temporal o fecha.

**Transformación**
- Estandarización de ids y fechas.
- Se crea 1 `record` por evento.

**Output**
- `RECORDS`:
  - `record_id` (determinístico o UUID)
  - `machine_id`
  - `start_date`, `end_date`, `source_work_order_id`
  - texto resumen si existe

**Gate**
- `machine_id` debe existir en `MACHINES`.

---

## Paso 2 — Descomposición del evento en trabajos (create JOBS)
**Input**
- Texto libre y/o líneas de tarea asociadas al record.

**Transformación**
- Se segmenta el contenido del record en unidades de trabajo:
  - por criterios del ERP (si viene desagregado), o
  - por chunking/reglas/LLM (si viene un bloque de texto)
- Se asigna scope técnico:
  - `jobs.system_id` y/o `jobs.subsystem_id`

**Output**
- `JOBS`:
  - `job_id`
  - `record_id`
  - `system_id` y/o `subsystem_id`
  - fechas si aplican, `job_type`

**Gates**
- `record_id` debe existir.
- Validación de scope:
  - `system_id IS NOT NULL OR subsystem_id IS NOT NULL`
- Gobernanza por máquina:
  - si hay `system_id`: debe existir en `MACHINE_SYSTEMS` para esa máquina.
  - si hay `subsystem_id`: su system debe estar en `MACHINE_SYSTEMS`.

---

## Paso 3 — Materialización de acciones (create ACTIONS)
**Input**
- Detalle del trabajo: qué se hizo realmente (inspección, cambio, ajuste, etc.).

**Transformación**
- Para cada job, se generan N acciones:
  - Se mapea `action_type` al catálogo `ACTION_TYPES`
  - Se define target técnico:
    - `actions.subsystem_id` (siempre)
    - `actions.component_id` (si disponible)

**Output**
- `ACTIONS`:
  - `action_id`, `job_id`, `action_type_id`
  - `subsystem_id` (required)
  - `component_id` (optional)
  - timestamp y notes

**Gates**
- `job_id` debe existir.
- `subsystem_id` obligatorio.
- Si `component_id` existe, debe pertenecer al subsystem.
- Consistencia con scope del job:
  - si job fijó subsystem, action debe usar ese subsystem.
  - si job fijó system, action.subsystem debe pertenecer a ese system.

---

## Paso 4 — Publicación y consumo
**Consumo típico BI**
- Métricas por máquina/sistema/subsystem:
  - recuento de acciones
  - frecuencia de reemplazos
  - backlog (records abiertos)
  - MTTR/tiempos (si hay timestamps)

**Consumo para alertas**
- Reglas se disparan sobre `ACTIONS` (no sobre texto).
  - Ejemplo: “más de X reemplazos en subsystem Y en 7 días”
  - Ejemplo: “acciones críticas en subsystem Z en turno nocturno”

**Vistas recomendadas (serving)**
- `vw_actions_enriched` (join ACTIONS + JOBS + RECORDS + catálogos)
- `fact_actions_daily` (agregados por día/máquina/subsystem/action_type)

---

## Trazabilidad
Trazabilidad mínima que debe quedar en Gold:
- `records.source_work_order_id`
- campos de auditoría (created_at, source_system, run_id si aplica)
- conservar `original_text` al menos a nivel record o staging

---

## Definición de Done (operativo)
- Puedo tomar un input crudo y poblar:
  - `RECORDS`, `JOBS`, `ACTIONS`
- Puedo responder preguntas de negocio sin parsing de texto.
- Las validaciones bloqueantes están definidas y automatizables.
