<div align="center">

# 🛰️ Argus — Agente Autónomo de Confiabilidad de Datos y Analítica

### Plataforma de datos agéntica que monitorea pipelines, se autodiagnostica ante incidentes y responde preguntas de negocio en lenguaje natural — construida sobre dbt, BigQuery y Claude.

<br>

![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-Transform%20%2B%20Tests-FF694B?style=for-the-badge&logo=dbt&logoColor=white)
![BigQuery](https://img.shields.io/badge/BigQuery-Warehouse-669DF6?style=for-the-badge&logo=googlebigquery&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-Sonnet%205-D97757?style=for-the-badge&logo=anthropic&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-Eval%20Gated-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)

<br>

> **Proof of Concept** — Argus simula el stack de confiabilidad de datos que una empresa data-driven necesita para pasar de apagar incendios de forma *reactiva* ("¿por qué cambió este número del dashboard?") a pipelines *proactivos* que se autodiagnostican, más una capa de analítica en lenguaje natural gobernada. Todo corre sobre datos sintéticos; no se usa información real de ninguna empresa.

<br>

[🎯 Problema](#-el-problema) · [💡 Solución](#-la-solución) · [📐 Arquitectura](#-arquitectura) · [🧱 Plataforma de Datos](#-plataforma-de-datos-dbt--bigquery) · [🧮 Capa Semántica](#-capa-semántica) · [🛡️ Guardrails](#️-guardrails--seguridad) · [🤖 Text-to-SQL](#-text-to-sql) · [🚨 Monitoreo](#-monitoreo-autónomo-capacidad-1) · [📊 Digest](#-digest-ejecutivo-capacidad-3) · [🧪 Evals](#-harness-de-evaluación) · [🏢 Config por Cliente](#-configuración-por-cliente) · [🚀 Quick Start](#-quick-start) · [🩺 Decisiones de Diseño](#-decisiones-de-diseño--tradeoffs) · [📁 Estructura](#-estructura-del-proyecto)

</div>

---

## 🎯 El Problema

Toda empresa que opera sobre datos choca con los mismos dos modos de falla a medida que escala:

| Síntoma | Impacto real |
|---|---|
| Los pipelines se rompen **en silencio** | Un KPI cambia y nadie sabe por qué hasta que un stakeholder se queja |
| El análisis de causa raíz es **manual** | Los analistas de guardia queman horas escribiendo queries ad-hoc |
| Los checks de calidad viven **dentro de un script** | Sin lineage, sin historial, sin gate — el dato malo llega al warehouse |
| Los analistas son un **cuello de botella** para preguntas | "¿Por qué bajaron las ventas la semana pasada?" espera en una fila |
| Los demos de LLM-sobre-SQL **alucinan joins** | Impresionan en la demo, fallan en producción |

---

## 💡 La Solución

Una plataforma de datos gobernada, con un agente encima que lee la capa *gobernada* — nunca tablas crudas:

```
Datos sintéticos multicanal  →  dbt (staging → marts + tests)  →  BigQuery
                                          │
                                 Capa semántica (métricas gobernadas)
                                          │
                                  Claude (Sonnet 5, vía API)
                                          │
                        ┌─────────────────┴─────────────────┐
                  Text-to-SQL en CLI                  Harness de evals
              (anclado a la capa semántica)       (precisión de ejecución)
```

1. **Modela** el warehouse con dbt — staging, marts, tests y freshness como ciudadanos de primera clase.
2. **Gobierna** las métricas en una capa semántica validada contra el SQL real de los modelos — no una lista de columnas mantenida a mano que se desactualiza sola.
3. **Cerca con guardrails** cualquier SQL generado antes de que toque BigQuery: solo lectura, sin PII, sin SQL apilado, con límite de filas.
4. **Responde** preguntas de negocio vía text-to-SQL, anclado a la capa semántica.
5. **Mide** la precisión del text-to-SQL con un harness de evals que compara resultados, no texto.
6. **Se configura por cliente** en un YAML — el código no cambia entre clientes con un stack similar.

---

## 📐 Arquitectura

Tres capas, cada una con una sola responsabilidad:

**Plataforma de datos (la fuente de verdad).** Un generador sintético parametrizado por cliente (`clients/*.yml`) escribe eventos multicanal a BigQuery. dbt transforma `raw → staging → marts`, con tests de datos y freshness declarada. Esta capa es determinista y totalmente testeable — el agente nunca inventa datos, lee tablas modeladas.

**Capa de agente (Claude vía API de Anthropic).** `argus/ask.py` recibe una pregunta, la ancla a la capa semántica, genera SQL, lo valida con guardrails, lo ejecuta con una cuenta de servicio de solo lectura, y devuelve una narrativa.

**Ingeniería transversal.** Guardrails, un harness de evals que actúa como gate de CI, y una capa de configuración por cliente envuelven ambas capas.

> **Estado:** las tres capacidades del diseño original están implementadas y verificadas con datos y credenciales reales — no solo en el papel. Integración con Jira/Slack/OpenClaw real y observabilidad de costo siguen en roadmap — ver [Roadmap](#-roadmap).

---

## 🧱 Plataforma de Datos (dbt + BigQuery)

El warehouse está modelado, no volcado:

```
models/
├── staging/      un modelo por fuente (stg_orders_web/app/pos), 1:1 con raw
├── intermediate/ union de los 3 canales, ephemeral (no crea tabla física)
└── marts/        mart_orders — una fila por orden, documentado, testeado
```

Todo vive en el dataset `argus_analytics`, sin importar la capa — hay un macro (`macros/generate_schema_name.sql`) que lo fuerza así, porque el comportamiento por defecto de dbt (`+schema: staging` → dataset nuevo `analytics_staging`) habría roto el modelo de permisos: la cuenta de solo lectura del agente solo tiene acceso a `argus_analytics`, no a datasets que dbt pudiera inventar.

```yaml
# models/staging/_staging.yml
sources:
  - name: raw
    freshness:
      warn_after:  { count: 6,  period: hour }
      error_after: { count: 12, period: hour }
    loaded_at_field: _ingested_at
    tables:
      - name: raw_orders_web
      - name: raw_orders_app
      - name: raw_orders_pos
```

`dbt build` corre modelos y tests en orden de dependencias. Tests reales, no de relleno: `revenue >= 0`, `ingestion_lag_minutes >= 0`, `order_id` único, `channel`/`status` acotados a valores válidos.

---

## 🧮 Capa Semántica

`semantic/metrics.yml` define `orders`, `revenue` y `cancellation_rate` — una sola vez. El agente recibe estas definiciones como contexto de anclaje y compone SQL a partir de métricas conocidas, no de aritmética de columnas inventada.

```yaml
metrics:
  - name: cancellation_rate
    calculation: "countif(status = 'cancelled') / count(order_id)"
    table: "mart_orders"
    grain: ["order_date", "channel", "country"]
```

**La validación no es cosmética.** `argus/semantic.py` parsea el SQL *real* de `mart_orders.sql` con `sqlglot` y confirma que cada métrica y cada dimensión de `grain` referencian columnas que existen de verdad — no una lista de columnas mantenida a mano que se desactualiza sola. La primera versión de este validador tenía un bug real: leía el `select * from final` final del modelo y se quedaba con el `*` literal en vez de resolver las columnas del CTE `final`. Se detectó rompiendo el mart a propósito antes de entregar el código — ver [Decisiones de Diseño](#-decisiones-de-diseño--tradeoffs).

---

## 🛡️ Guardrails & Seguridad

El agente habla con BigQuery a través de una cuenta de servicio **de solo lectura** (`sa-agent`), separada de la que carga datos (`sa-loader`) desde la Fase 0. Cada query generada se valida *antes* de ejecutarse:

```python
# argus/guardrails/sql.py
BLOCKED_STATEMENT_TYPES = (exp.Insert, exp.Update, exp.Delete, exp.Drop, exp.Create, exp.Alter, exp.Merge)

def validate(sql: str, max_rows: int = 10_000, ...) -> str:
    statements = [s for s in sqlglot.parse(sql, read="bigquery") if s is not None]
    if len(statements) > 1:
        raise GuardrailViolation("SQL apilado no permitido.")   # ver nota abajo
    ...
```

Defensa en profundidad:

- **IAM de solo lectura** — el control primario. `sa-agent` no *puede* escribir, sin importar qué SQL genere.
- **Solo `SELECT`, sin `SELECT *`** — columnas nombradas explícitamente.
- **Rechazo de SQL apilado** — `sqlglot.parse_one()` descarta en silencio cualquier sentencia después de un `;`; un ataque tipo `SELECT ...; DROP TABLE ...;` se vería "limpio" si solo se mirara la primera. Este proyecto usa `sqlglot.parse()` (plural) y exige exactamente una sentencia. Encontrado probando casos adversarios antes de escribir los tests, no después de un incidente.
- **Columnas PII bloqueadas** — lista explícita, inglés + español, coincidencia exacta sobre nombre normalizado (no heurística difusa: en seguridad, predecible le gana a "inteligente").
- **`LIMIT` forzado y acotado** — un `LIMIT` ausente se agrega; uno que excede el máximo se recorta. Ninguno de los dos puede saltarse el cap de filas.
- **Cap de bytes facturados** — `maximum_bytes_billed` a nivel de job de BigQuery; una query mala no puede escanear (ni facturar) el warehouse completo.

---

## 🤖 Text-to-SQL

```bash
$ python -m argus.ask "¿cuántas órdenes hubo en total por canal?" --show-sql

SQL generado:
SELECT channel, COUNT(order_id) AS orders
FROM `argus-data-agent.argus_analytics.mart_orders`
GROUP BY channel

channel  orders
    pos   39503
    app   87303
    web  131325

El canal web concentra la mayor parte de las órdenes con 131,325 (~51% del
total), seguido de app con 87,303 (34%) y pos con 39,503 (15%)...
```

Flujo (`argus/ask.py`): pregunta → `generate_sql()` (Claude + contexto semántico) → `validate()` (guardrails) → `run_query()` (BigQuery, cuenta de solo lectura) → `generate_narrative()` (Claude, español). Construido primero como CLI a propósito — cualquier transporte futuro (Slack, etc.) llamaría a estas mismas funciones.

---

## 🚨 Monitoreo Autónomo (Capacidad 1)

Un monitor que corre checks de calidad sobre `mart_pipeline_health` (volumen, frescura, tasa de nulos — z-score contra un baseline de 7 días, no umbrales fijos adivinados) y, cuando algo falla, le pide a Claude un diagnóstico en lenguaje llano:

```
======================================================================
[ERROR] freshness — canal 'web'
======================================================================
'web' no recibe datos hace 22.9h (umbral error: 12h)

Detectado:  El canal 'web' no recibe datos desde hace 22.9h, superando el
            umbral de error (12h).
Diagnóstico: causa no determinada desde los datos disponibles; puede
            tratarse de un corte en el pipeline de ingesta, un fallo en
            la fuente origen, o un problema de scheduling del job.
Acción:     Revisar el estado del job/pipeline de ingesta (logs,
            orquestador) y confirmar si la fuente origen está emitiendo.
======================================================================
```

Nótese lo que el diagnóstico **no** hace: no inventa una causa específica que los datos no sustentan. El prompt lo instruye explícitamente a decir "causa no determinada" en vez de adivinar — verificado con datos reales, no solo prometido en el prompt.

**Notificación desacoplada a propósito.** `argus/notifiers.py` define un `Protocol` (`ConsoleNotifier` hoy); Jira/Slack implementarían la misma interfaz sin tocar la lógica de detección o diagnóstico. Ver [Decisiones de Diseño](#-decisiones-de-diseño--tradeoffs) para por qué `mart_pipeline_health` existe en vez de darle a `sa-agent` acceso a `argus_raw`.

```bash
python -m argus.monitors.run
```

---

## 📊 Digest Ejecutivo (Capacidad 3)

Un brief periódico que calcula las métricas gobernadas para la semana más reciente vs. la anterior — anclado a la **fecha más reciente que existe en los datos**, no al calendario real (los datos son un snapshot generado una vez, no un pipeline vivo; anclar a "hoy" produciría comparaciones contra semanas vacías).

Diferencia de arquitectura con la Capacidad 2: aquí el SQL sale directo de `calculation` en la capa semántica — determinista, sin pasar por un LLM. Claude solo escribe la narrativa a partir de números ya calculados, nunca decide qué calcular; un digest programado necesita el mismo número cada vez que corre sobre los mismos datos.

```
📊 Digest ejecutivo — semana terminando 2026-07-15

- Órdenes: 20046.00 (semana previa: 19984.00, cambio: +0.3%)
- Ingresos: 902193.82 (semana previa: 899061.01, cambio: +0.3%)
- Tasa de cancelación: 0.08 (semana previa: 0.08, cambio: -3.7%)

Las métricas principales se mantuvieron estables esta semana... no se
observan variaciones significativas que requieran atención inmediata.
```

```bash
python -m argus.digest
```

Evaluado por **precisión de ejecución** (¿coinciden los result sets?), no por coincidencia de texto de SQL — dos queries distintas pueden ser igualmente correctas.

```yaml
# argus/evals/cases.yml
- id: cancellation_rate_overall
  question: "¿Cuál es la tasa de cancelación general, como proporción?"
  reference_sql: "references/cancellation_rate_overall.sql"
```

Las 6 queries de referencia semilla se verificaron dos veces antes de confiar en ellas: sintaxis con `sqlglot`, y lógica recalculando los mismos agregados con `pandas` directo desde el generador determinista (mismo seed que los datos reales), de forma independiente a BigQuery.

> **Precisión de evals (corrida real, 16 jul 2026):** **6/6 (100%)** sobre la suite semilla.
> Nota honesta: `n=6` es una muestra pequeña — suficiente para probar que el mecanismo de comparación funciona, no una garantía estadística. Ampliar la cobertura es trabajo de roadmap, no un hueco oculto.

El runner clasifica cada falla por tipo (`result_mismatch`, `guardrail_rejected`, `invalid_sql`, `generation_error`, `reference_query_failed`) para que un fallo diga *por qué*, no solo *que* falló. Gate obligatorio de CI (`--min-accuracy 0.90`).

> **Costo por pregunta:** aún no instrumentado (requiere la capa de observabilidad de OpenTelemetry, en roadmap). Sonnet 5 cuesta $2/$10 por millón de tokens de entrada/salida (precio introductorio vigente a jul-2026); cada llamada de `ask()` usa del orden de 1,000-2,000 tokens totales, es decir, fracciones de centavo por pregunta.

---

## 🏢 Configuración por Cliente

Canales, países, mezclas y volúmenes del generador **no** están en el código — viven en `clients/<nombre>.yml`, validados con `pydantic` (los pesos de país/status deben sumar 1.0, o falla temprano con un mensaje claro).

```yaml
# clients/example.yml
channels:
  web: { base_volume: 1400, weekend_multiplier: 1.15 }
  app: { base_volume: 900,  weekend_multiplier: 1.25 }
  pos: { base_volume: 500,  weekend_multiplier: 0.6 }
countries: { MX: 0.45, US: 0.30, CO: 0.15, ES: 0.10 }
```

Para un cliente nuevo con un stack similar: se escribe un YAML nuevo, el Python no se toca. Verificado generando datos completos para un cliente ficticio (`acme-corp`, canal `marketplace`, países BR/AR) sin editar una sola línea de código.

---

## 🚀 Quick Start

### Prerrequisitos

```bash
Python 3.12 (NO 3.14 -- protobuf/dbt no compilan ahí, ver nota abajo)
Un proyecto de BigQuery + dos service accounts (sa-loader, sa-agent) -- ver docs/setup-gcp.md
Una API key de Anthropic -- console.anthropic.com/settings/keys
```

> **Nota de compatibilidad:** este proyecto se desarrolló y probó contra un entorno con Python 3.14 preinstalado, donde `protobuf` (dependencia de dbt y de google-cloud-bigquery) falla al compilar con `TypeError: Metaclasses with custom tp_new are not supported`. La solución fue crear el entorno virtual con Python 3.12 explícito (`python3.12 -m venv .venv`). Si `dbt --version` o `import google.protobuf` fallan con ese error, es esto.

### Setup

```bash
cp .env.example .env      # GCP_PROJECT_ID, rutas a llaves, ANTHROPIC_API_KEY
mkdir -p ~/.dbt && cp profiles.example.yml ~/.dbt/profiles.yml
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m scripts.bootstrap_bq       # crea datasets, verifica permisos IAM
```

### Generar datos y construir el warehouse

```bash
python -m data.synthetic.generate_data --days 90     # ~258k filas sintéticas
set -a && source .env && set +a                       # exporta env vars para dbt
dbt deps && dbt build
```

### Usar el agente

```bash
python -m argus.ask "¿qué canal generó más ingresos?" --show-sql
python -m argus.evals.run                              # harness completo
```

---

## 🩺 Decisiones de Diseño & Tradeoffs

*Lo senior no es la lista de features — es por qué cada pieza está construida como está, y qué se rompió en el camino.*

- **`generate_schema_name` sobreescrito para forzar un solo dataset.** El comportamiento por defecto de dbt con `+schema: staging` crea datasets nuevos (`argus_analytics_staging`). El permiso de lectura de `sa-agent` (Fase 0) está scopeado a `argus_analytics` únicamente — sin el override, dbt habría creado infraestructura fuera de ese permiso sin que nadie lo notara hasta que el agente intentara leer ahí.

- **Validación de la capa semántica contra el SQL real, con un bug real encontrado en el camino.** La primera versión de `_output_columns()` leía el `select * from final` de cierre de cada modelo y se quedaba con el `*` literal — resultado: **todo pasaba, incluso lo que no debía**, el peor tipo de bug en un validador. Se encontró corriendo la suite antes de entregar, se arregló para resolver la cadena de CTEs, y se confirmó rompiendo el mart a propósito (un typo deliberado en `revenue`) para probar que el test de verdad lo atrapa.

- **Guardrails que rechazan SQL apilado explícitamente.** `sqlglot.parse_one()` descarta en silencio todo después de un `;`. Confiar en eso habría dejado un hueco de seguridad real (`SELECT ...; DROP TABLE ...;` se vería limpio). Se encontró probando el caso adversario *antes* de escribir el resto del validador, no después.

- **`create_bqstorage_client=False` en el ejecutor de queries.** El camino "rápido" de BigQuery requiere `bigquery.readsessions.create` a nivel de *proyecto* — un permiso que no se puede acotar a un dataset como sí se hizo con `dataViewer`. Con `SQL_MAX_ROWS` topando el tamaño del resultado, la velocidad extra es irrelevante; mantener a `sa-agent` sin permisos de proyecto no lo es.

- **Precisión de ejecución, no coincidencia de string, como métrica de eval.** Dos queries distintas pueden ser igualmente correctas. Se sigue evaluando si los *result sets* coinciden, normalizando orden de filas/columnas antes de comparar.

- **Configuración por cliente desde la Fase 1, no como refactor tardío.** Canales/países/volúmenes en YAML en vez de constantes en Python — la pregunta de qué tan fácil sería replicar este proyecto para un cliente real llevó a esta decisión antes de que hiciera falta, no después.

- **Solo lectura por construcción, no por política.** La capa de guardrails es real, pero el control primario es que el rol IAM del agente *no puede* escribir.

---

## ⚠️ Limitaciones Conocidas

- Corre sobre **datos sintéticos**; las distribuciones son plausibles pero no reales.
- El **tier sandbox de BigQuery** tiene límite de retención de 60 días y restricciones de DML (que este proyecto no necesita — ver `docs/setup-gcp.md`).
- Solo se generan 3 canales (web/app/pos), no los 6 originalmente contemplados (email/chat/social quedan fuera de alcance de este PoC).
- El CI corre `dbt build` + evals contra el **mismo** dataset `argus_analytics` de desarrollo local, no uno aislado — ver la nota en `.github/workflows/ci.yml`.
- Costo por pregunta aún no instrumentado (pendiente de capa de observabilidad).
- El diagnóstico de incidentes es una sola llamada a Claude con los números del finding, no una ronda de queries de seguimiento adicionales — suficiente para el PoC, no la versión más elaborada del "agente investiga por su cuenta".
- Notificación de incidentes vía consola (`ConsoleNotifier`); Jira/Slack reales no están conectados todavía, aunque la interfaz (`Notifier` Protocol) ya está lista para recibirlos sin cambios en la lógica.
- Integración con OpenClaw (orquestación programada real) no implementada: los comandos se corren manualmente.

---

## 📦 Tech Stack

| Capa | Tecnología | Propósito |
|---|---|---|
| Lenguaje | Python 3.12 | Generadores, agente, evals, guardrails |
| Warehouse | Google BigQuery | Warehouse en la nube (sandbox) |
| Transformación | dbt-core 1.8 | Modelos, tests, freshness, lineage |
| Razonamiento | Claude Sonnet 5 (API de Anthropic) | Text-to-SQL, narrativa |
| Validación de config | pydantic v2 | Config por cliente, capa semántica |
| Guardrails de SQL | sqlglot | Parsea/valida cada query generada |
| CI/CD | GitHub Actions | Lint + dbt build + gate de evals en cada PR |

---

## 🔁 CI/CD

Dos jobs: `lint` (ruff + pytest, sin red real) y `build_and_eval` (dbt build + harness de evals, gate en `--min-accuracy 0.90`), que corre solo si `lint` pasa.

```yaml
build_and_eval:
  needs: lint
  steps:
    - run: dbt build
    - run: python -m argus.evals.run --min-accuracy 0.90
```

Requiere 3 GitHub Secrets: `SA_LOADER_KEY_JSON`, `SA_AGENT_KEY_JSON`, `ANTHROPIC_API_KEY`.

---

## 📁 Estructura del Proyecto

```
argus-data-agent/
│
├── 📄 .env.example · docker-compose.yaml · Dockerfile · Makefile
├── 📄 dbt_project.yml · packages.yml · profiles.example.yml
│
├── 📂 docs/
│   └── setup-gcp.md                  ← dos service accounts, permisos IAM
│
├── 📂 data/synthetic/
│   └── generate_data.py              ← generador multicanal, --inject-fault
│
├── 📂 clients/
│   └── example.yml                   ← config por cliente (canales, países, mezclas)
│
├── 📂 models/                        ← dbt: staging → intermediate → marts
├── 📂 macros/
│   └── generate_schema_name.sql      ← fuerza un solo dataset
│
├── 📂 semantic/
│   └── metrics.yml                   ← métricas gobernadas
│
├── 📂 argus/
│   ├── config.py · clients.py        ← settings, credenciales por rol
│   ├── client_config.py              ← loader/validador de clients/*.yml
│   ├── semantic.py                   ← loader/validador de metrics.yml
│   ├── warehouse.py                  ← único punto de acceso a BigQuery
│   ├── ask.py                        ← text-to-SQL (CLI, Capacidad 2)
│   ├── digest.py                     ← digest ejecutivo (Capacidad 3)
│   ├── notifiers.py                  ← Notifier Protocol (Console hoy, Jira/Slack después)
│   ├── guardrails/sql.py             ← validador de SQL
│   ├── monitors/
│   │   ├── checks.py                 ← freshness/volumen/nulls (Capacidad 1)
│   │   └── run.py                    ← orquestador: checks → diagnóstico → notify
│   └── evals/
│       ├── cases.yml · references/*.sql
│       └── run.py                    ← scorer de precisión de ejecución
│
├── 📂 tests/                          ← 124 tests, sin llamadas reales a red
│
└── 📂 .github/workflows/
    └── ci.yml                        ← lint + dbt build + gate de evals
```

---

## 🗺️ Roadmap

```
✅ Fase 0-1   ▸  Cimientos, IAM, generador sintético + config por cliente
✅ Fase 2-3   ▸  Modelos dbt + capa semántica validada
✅ Fase 4-5   ▸  Guardrails + text-to-SQL funcionando con datos reales
✅ Fase 6     ▸  Harness de evals: 6/6 (100%) + gate de CI activo
✅ Fase 7     ▸  Monitoreo autónomo (Capacidad 1): freshness/volumen/nulls + diagnóstico
✅ Fase 9     ▸  Digest ejecutivo (Capacidad 3)

Próximo   ▸  Observabilidad: tokens/costo por corrida
          ▸  Notifiers reales: JiraNotifier, SlackNotifier (mismo Protocol)
          ▸  Orquestación programada (OpenClaw o cron)

Futuro    ▸  Ampliar cobertura de evals más allá de 6 casos semilla
          ▸  Aislar CI en su propio dataset (argus_ci) con fixtures propias
          ▸  Investigación multi-paso en el monitor (follow-up queries, no solo 1 diagnóstico)
          ▸  Soporte multi-warehouse (Snowflake/DuckDB)
```

---

## 📄 Licencia

MIT License — libre para uso educativo y de portafolio.

---

<div align="center">

**Desarrollado como Proof of Concept de confiabilidad de datos autónoma y analítica self-service.**

*Utiliza datos completamente sintéticos generados para fines demostrativos. No contiene información real de ninguna empresa.*

</div>
