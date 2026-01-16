# docs/04_quality_gates.md

# Quality Gates y controles (para que el producto no se degrade)

## Objetivo
Evitar que el pipeline produzca data inconsistente o inútil. Esto se logra con:
- reglas bloqueantes (fail fast)
- métricas de observabilidad (warn + trending)
- reconciliación de volúmenes

---

## 1) Gates bloqueantes (ERROR)

### Catálogo y jerarquía
- `subsystems.system_id` existe en `systems`
- `components.subsystem_id` existe en `subsystems`

### Configuración por máquina
- `machine_systems.machine_id` existe en `machines`
- `machine_systems.system_id` existe en `systems`
- PK (`machine_id`, `system_id`) sin duplicados

### Transaccional básico
- `records.machine_id` existe en `machines`
- `jobs.record_id` existe en `records`
- `actions.job_id` existe en `jobs`
- `actions.action_type_id` existe en `action_types`

### Scope y target (reglas del negocio)
- JOB: `system_id IS NOT NULL OR subsystem_id IS NOT NULL`
- ACTION: `subsystem_id IS NOT NULL`

### Consistencia jerárquica
- Si `actions.component_id` no es null:
  - `components.component_id` pertenece a `actions.subsystem_id`
- Si `jobs.subsystem_id` no es null:
  - `subsystems.subsystem_id` es consistente con `jobs.system_id` si ambos existen

### Gobernanza por máquina
- Si `jobs.system_id` existe:
  - debe existir (`records.machine_id`, `jobs.system_id`) en `machine_systems`
- Si `jobs.subsystem_id` existe:
  - el `system_id` del subsystem debe existir en `machine_systems` para esa máquina

---

## 2) Métricas de calidad (WARN)
Estas no bloquean al inicio, pero se monitorean.

### Volúmenes
- records por semana (drift)
- jobs/record promedio
- actions/job promedio

### Completitud
- % records sin end_date (backlog)
- % jobs con solo system_id (sin subsystem)
- % actions sin component_id (normal al inicio, pero monitorear)

### Texto/Parsing (si aplica)
- longitud promedio de textos
- ratio de tokens no alfabéticos
- top keywords desconocidas (para mejorar diccionarios)

---

## 3) Reconciliación (sanity checks)
- `count(actions)` debe ser coherente con `count(jobs)` y `count(records)` (no explosiones)
- Por record:
  - un record con 0 jobs es inválido
  - un job con 0 actions es inválido (salvo estado “planificado”, si lo defines)

---

## 4) Niveles de enforcement (rollout práctico)
- Semana 1: gates críticos en **WARN** mientras se calibra el parsing.
- Semana 2+: FKs + scope/target en **ERROR**.
- Semana 3+: gobernanza por máquina (`machine_systems`) pasa a **ERROR**.

---

## 5) Observabilidad operativa (mínimo)
Registrar por corrida:
- `run_id`, fuente, rango temporal
- rowcounts por tabla
- cantidad de errores por tipo de gate
- top 10 causas (ej: machine_id desconocido, subsystem inválido)

Recomendación: tabla `pipeline_runs` + `pipeline_run_errors`.
