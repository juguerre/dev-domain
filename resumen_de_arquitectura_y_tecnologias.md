# Resumen Ejecutivo: Arquitectura de Data Streaming Basada en Eventos

Este documento sintetiza los temas tratados, las decisiones de diseño adoptadas, las conclusiones alcanzadas y el cuadro comparativo de estándares y tecnologías para el desarrollo del **Sistema y Catálogo de Dominio de Data Streaming**.

---

## 1. Resumen de Temas Tratados y Conclusiones Alcanzadas

### Objetivo y Arquitectura General
* **Catálogo de Dominio Centralizado:** Sistema orientado a eventos de lectura mayoritaria, diseñado para unificar entidades multisistema (como componentes, proyectos, aplicaciones, etc.).
* **Flujo End-to-End:** Soporta ingesta de fuentes nativas (eventos) y fuentes *legacy* (vía archivos planos o CDC), almacenamiento inmutable en capa *raw* (*System of Record for Inputs*), procesamiento/validación en la puerta de entrada (*Ingestion Gate*), almacenamiento canónico híbrido (`Core` + `JSONB`), proyecciones (grafos/calidad) y una fachada *legacy* de salida.

### Ingesta Raw e Inmutabilidad (*Raw-First*)
* **Conclusión:** Se adopta el patrón *Raw-First*. La tabla `raw_event` (almacenada en PostgreSQL/AlloyDB) actúa como el origen inmutable e inalterable.
* **Garantías:** Proporciona trazabilidad técnica, auditoría completa, capacidad de reescritura/reprocesamiento histórico (*replay*) e independiza el ciclo de vida de los datos de la retención temporal de la capa de transporte (Kafka/Pub/Sub).

### Estrategia para Fuentes Legacy (Ficheros Diarios y CDC)
* **Detección de Cambios (*Diffing*):** Para archivos diarios de sistemas *legacy*, se aplican mecanismos de comparación frente al estado anterior.
* **Conclusión sobre SCD Tipo 2:** Se concluye que almacenar las capturas históricas mediante un modelo de dimensión de cambio lento **SCD2** (con campos `valid_from`, `valid_to`, `is_current` e índice parcial sobre la versión activa) permite realizar *diffing* eficiente usando hashes SHA-256. Esto dispara de forma determinista eventos de mutación (`.created`, `.patched`, `.deleted`) e incorpora *circuit breakers* para mitigar fallos masivos por archivos corruptos o vacíos.

### Gobernanza, Contratos y Esquema Extensible
* **Envoltura Estándar:** Todos los eventos adoptan la especificación **CloudEvents (v1.0)** para desacoplar metadatos contextuales del *payload*.
* **Modelo Híbrido Canónico:** La entidad canónica se estructura en:
  * **Core:** Contrato fuerte fuertemente tipado en columnas físicas.
  * **JSONB (`extensions`):** Bloque flexible de tipado dinámico delimitado por espacios de nombres (*namespaces*) por origen.
* **Evolución sin Breaking Changes:** Los cambios aditivos en extensiones no rompen compatibilidad. Las extensiones pueden "ascender" al *Core* mediante un comité de gobernanza (RFC).

### Pipeline de Procesamiento e *Ingestion Gate*
* El *Ingestion Gate* realiza la validación sintáctica (envoltura CloudEvents y JSON Schema), hidratación/fusión en memoria (*In-Memory Merge*) y evaluación de reglas de negocio/calidad.
* **Reglas Bloqueantes (*BLOCKING*):** Mueven el evento a la tabla `quarantine` interrumpiendo la consolidación canónica.
* **Reglas No Bloqueantes (*WARNING*):** Aceptan e integran el evento en la base canónica, pero reducen el *Quality Score* de la entidad, registrando de forma atómica un evento append-only en `quality_event`.

### Selección de Cómputo para el Gate (Microservicio vs. Apache Beam)
* **Conclusión:** Para **volúmenes bajos/medios y lógica no altamente compleja**, la opción más aconsejable, robusta y ACID-compliant es un **microservicio/worker estático**.
* **Manejo del Cursor:** El patrón más eficiente en PostgreSQL es el uso de **`FOR UPDATE SKIP LOCKED`** sobre el estado de procesamiento de la fila (`processing_status`). Permite un escalado horizontal transparente de múltiples trabajadores en paralelo, garantizando distribución atómica sin contención.
* **Apache Beam / Flink:** Se reserva para procesado distribuido de gran escala, agregaciones complejas en ventanas de tiempo (*Windowing*) o métricas en tiempo real a alta velocidad.

### Gestión de Concurrencia y Eventos Desordenados
* **Transporte:** Uso de *Partition Key* por `catalog_id` / `subject` en la capa de transporte para asegurar procesamiento ordenado FIFO.
* **Base de Datos:** Comparación de marcas de tiempo monotónicas (`time` de CloudEvent vs. `last_event_time` en BBDD) para descartar o alertar sobre *stale updates*. Uso de *Source-Scoped Merging* sobre JSONB para lograr operaciones de actualización conmutativas entre orígenes independientes.

---

## 2. Cuadro de Estándares y Tecnologías (Escenario Agnóstico vs. Escenario GCP)

| Capa / Subsistema | Función / Patrón Aplicado | Escenario Agnóstico / Open Source (Multi-cloud) | Equivalente Nativo / Gestionado en GCP |
| :--- | :--- | :--- | :--- |
| **Estándar de Evento** | Envoltura común de metadatos contextuales | **CloudEvents v1.0** (CNCF) | **CloudEvents** (soporte nativo en Eventarc) |
| **Gobernanza de Esquemas** | Registro y contratos de validación (JSON Schema, Avro, Protobuf) | **Apicurio Registry** / Confluent Schema Registry / JSON Schema Draft 2020-12 | **Pub/Sub Schema Registry** / Artifact Registry |
| **Transporte de Eventos** | Bus distribuido / Streaming log append-only | **Apache Kafka** / Apache Pulsar | **Cloud Pub/Sub** / **Managed Service for Apache Kafka** |
| **Enrutamiento Event-Driven** | Disparo de eventos y orquestación sin servidor | **Knative Eventing** / CloudEvents SDK | **Eventarc** (Router serverless basado en CloudEvents) |
| **Adaptación / CDC Legacy** | Captura de deltas de BBDDs o archivos batch | **Debezium** / **Apache NiFi** / Custom Cron Workers | **Datastream** (CDC gestionado) / **Cloud Storage + Eventarc** |
| **Almacenamiento Raw & Canónico** | BBDD relacional inmutable + Core + JSONB | **PostgreSQL** (con JSONB e índices GIN) | **AlloyDB for PostgreSQL** / Cloud SQL for PostgreSQL |
| **Procesamiento & Gate Worker** | Ingestion Gate, validación, merge & QA scoring | **Microservicios (Go, Java, Python)** con `FOR UPDATE SKIP LOCKED` / **Apache Beam** (en Flink/Spark) | **Cloud Run** (Microservicios) / **Cloud Dataflow** (Apache Beam) |
| **Proyección de Grafos** | Análisis de impacto y relaciones transitivas | **Neo4j** (Community / Enterprise) | **Neo4j AuraDB on GCP Marketplace** / **Spanner Graph** |
| **Proyección de Calidad / Log** | Log append-only y agregaciones de score | Tablas en **PostgreSQL** (`quality_event`, `entity_quality_current`) | **AlloyDB** / **BigQuery** (para analítica histórica) |
| **Fachada Legacy & APIs** | Exposición REST y generación dual de ficheros batch | APIs en **FastAPI** / **Spring Boot** + MinIO + Workers batch | **Cloud Run** + **Apigee** (APIs) / **Cloud Run Jobs** + **Cloud Storage** (Batch) |