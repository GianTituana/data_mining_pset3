# 🚕 NYC Taxi Data Pipeline - Proyecto 03 (Gian Tituaña)
## Data Mining | Universidad San Francisco de Quito

Pipeline de ingesta y análisis de datos de NYC TLC Trip Records (2015-2025) utilizando Spark, Jupyter y Snowflake con arquitectura One Big Table (OBT).

---

## 📋 Tabla de Contenidos

- [Arquitectura del Sistema](#-arquitectura-del-sistema)
- [Matriz de Cobertura de Datos](#-matriz-de-cobertura-de-datos)
- [Instalación y Configuración](#-instalación-y-configuración)
- [Variables de Entorno](#-variables-de-entorno)
- [Ejecución del Pipeline](#-ejecución-del-pipeline)
- [Diseño de Esquemas](#-diseño-de-esquemas)
- [Calidad y Auditoría](#-calidad-y-auditoría)
- [Preguntas de Negocio](#-preguntas-de-negocio)
- [Troubleshooting](#-troubleshooting)
- [Checklist de Aceptación](#-checklist-de-aceptación)

---

## 🏗️ Arquitectura del Sistema

### Diagrama de Flujo
```
┌─────────────────────────────────────────────────────────────────┐
│                      NYC TLC DATA SOURCE                         │
│            https://d37ci6vzurychx.cloudfront.net/               │
│                   (Parquet Files 2015-2025)                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ HTTP Download
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DOCKER COMPOSE ENVIRONMENT                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           JUPYTER + PYSPARK NOTEBOOK                      │  │
│  │                                                            │  │
│  │  Notebooks:                                                │  │
│  │  ├── 01_ingesta_parquet_raw.ipynb                         │  │
│  │  ├── 02_enriquecimiento_y_unificacion.ipynb              │  │
│  │  ├── 03_construccion_obt.ipynb                            │  │
│  │  ├── 04_validaciones_y_exploracion.ipynb                 │  │
│  │  └── 05_data_analysis.ipynb                              │  │
│  │                                                            │  │
│  │  Ports: 8888 (Jupyter), 4040 (Spark UI)                  │  │
│  └────────────────────┬──────────────────────────────────────┘  │
└───────────────────────┼──────────────────────────────────────────┘
                        │
                        │ JDBC Connection
                        │ (Snowflake Connector)
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        SNOWFLAKE                                 │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  DATABASE: NY_TAXI                                      │    │
│  │                                                          │    │
│  │  ┌────────────────────────────────────────────────┐    │    │
│  │  │  SCHEMA: RAW (Staging Layer)                   │    │    │
│  │  │                                                 │    │    │
│  │  │  Tables:                                        │    │    │
│  │  │  ├── yellow_trips_YYYY_MM                      │    │    │
│  │  │  ├── green_trips_YYYY_MM                       │    │    │
│  │  │  ├── taxi_zone_lookup                          │    │    │
│  │  │  └── audit_log                                 │    │    │
│  │  │                                                 │    │    │
│  │  │  Metadata: run_id, source_year, source_month,  │    │    │
│  │  │            ingested_at_utc, service_type       │    │    │
│  │  └────────────────────────────────────────────────┘    │    │
│  │                          │                              │    │
│  │                          │ Enrichment & Unification     │    │
│  │                          ▼                              │    │
│  │  ┌────────────────────────────────────────────────┐    │    │
│  │  │  SCHEMA: ANALYTICS (Consumption Layer)         │    │    │
│  │  │                                                 │    │    │
│  │  │  Table: obt_trips (One Big Table)              │    │    │
│  │  │                                                 │    │    │
│  │  │  Grain: 1 row = 1 trip                         │    │    │
│  │  │                                                 │    │    │
│  │  │  Columns (70+):                                │    │    │
│  │  │  ├── Time Dimensions (10)                      │    │    │
│  │  │  ├── Location Dimensions (6)                   │    │    │
│  │  │  ├── Service & Codes (8)                       │    │    │
│  │  │  ├── Trip Facts (3)                            │    │    │
│  │  │  ├── Fare Components (9)                       │    │    │
│  │  │  ├── Derived Metrics (3)                       │    │    │
│  │  │  └── Lineage Metadata (5)                      │    │    │
│  │  └────────────────────────────────────────────────┘    │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Componentes del Sistema

| Componente | Tecnología | Propósito | Puerto |
|-----------|-----------|-----------|--------|
| **Ingesta & Procesamiento** | PySpark 3.5 | Lectura de Parquet, transformaciones, carga | - |
| **Ambiente de Desarrollo** | Jupyter Notebook | Notebooks interactivos para ETL | 8888 |
| **Monitoreo Spark** | Spark UI | Monitoreo de jobs y performance | 4040 |
| **Data Warehouse** | Snowflake | Almacenamiento persistente (RAW + ANALYTICS) | Cloud |
| **Orquestación** | Docker Compose | Gestión de contenedores y red | - |

### Flujo de Datos

1. **Extracción**: Descarga de archivos Parquet desde NYC TLC (2015-2025)
2. **Staging (RAW)**: Carga incremental mes a mes con metadatos de lineage
3. **Enriquecimiento**: Join con Taxi Zone Lookup, normalización de catálogos
4. **Transformación**: Cálculo de métricas derivadas (duración, velocidad, tip_pct)
5. **OBT Construction**: Consolidación en tabla desnormalizada única
6. **Validación**: Controles de calidad (rangos, nulos, coherencia)
7. **Analytics**: Consultas de negocio sobre OBT

---

## 📊 Matriz de Cobertura de Datos

### Cobertura 2015-2025 (Yellow Taxi)

| Año | Ene | Feb | Mar | Abr | May | Jun | Jul | Ago | Sep | Oct | Nov | Dic | Total |
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-------|
| 2015 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2016 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2017 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2018 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2019 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2020 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2021 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2022 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2023 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2024 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2025 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⏳ | ⏳ | ⏳ | ⏳ | 8/12 |

### Cobertura 2015-2025 (Green Taxi)

| Año | Ene | Feb | Mar | Abr | May | Jun | Jul | Ago | Sep | Oct | Nov | Dic | Total |
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-------|
| 2015 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2016 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2017 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2018 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2019 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2020 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2021 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2022 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2023 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2024 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 12/12 |
| 2025 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⏳ | ⏳ | ⏳ | ⏳ | 8/12 |

**Leyenda:**
- ✅ = Datos cargados exitosamente
- ❌ = Archivo no disponible o falló la carga
- ⏳ = Datos aún no disponibles (futuro)


## 🚀 Instalación y Configuración

### Pre-requisitos

- Docker Engine 20.10+
- Docker Compose 2.0+
- 8GB RAM mínimo (16GB recomendado)
- 50GB espacio en disco
- Cuenta de Snowflake activa

### Paso 1: Clonar el Repositorio
```bash
git clone https://github.com/GianTituana/data_mining_pset3.git
cd nyc-taxi-pipeline
```

### Paso 2: Configurar Variables de Entorno
```bash
# Copiar plantilla
cp .env.example .env

# Editar con tus credenciales
nano .env  # o usar tu editor favorito
```

### Paso 3: Estructura de Directorios
```
nyc-taxi-pipeline/
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── README.md
├── notebooks/
│   ├── 01_ingesta_parquet_raw.ipynb
│   ├── 02_enriquecimiento_y_unificacion.ipynb
│   ├── 03_construccion_obt.ipynb
│   ├── 04_validaciones_y_exploracion.ipynb
│   └── 05_data_analysis.ipynb
├── work/
│   └── (archivos temporales de Spark)
│   └── (notebooks ejecutados sin limpieza de outpus)
└── evidencias/
    ├── evidencias.md
```

### Paso 4: Levantar la Infraestructura
```bash
# Iniciar contenedores
docker-compose up -d

# Verificar que estén corriendo
docker-compose ps

# Ver logs
docker-compose logs -f pyspark-notebook
```

### Paso 5: Acceder a Jupyter


1. Abrir en navegador:
```
http://localhost:8888
```

### Paso 6: Verificar Spark UI
```
http://localhost:4040
```

---

## 🔐 Variables de Entorno

### Archivo `.env.example` (Plantilla sin credenciales)
```bash
# ===================================
# SNOWFLAKE CONFIGURATION
# ===================================
SNOWFLAKE_ACCOUNT=tu-cuenta.snowflakecomputing.com
SNOWFLAKE_USER=tu_usuario
SNOWFLAKE_PASSWORD=tu_contraseña
SNOWFLAKE_DATABASE=NY_TAXI
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_SCHEMA_RAW=RAW
SNOWFLAKE_SCHEMA_ENRICHED=ENRICHED
SNOWFLAKE_SCHEMA_ANALYTICS=ANALYTICS


# ===================================
# DATA SOURCE
# ===================================
PARQUET_BASE_URL=https://d37ci6vzurychx.cloudfront.net/trip-data
START_YEAR=2015
END_YEAR=2025
SERVICES=yellow,green

### Descripción de Variables

| Variable | Descripción | Ejemplo | Obligatoria |
|----------|-------------|---------|-------------|
| **SNOWFLAKE_ACCOUNT** | URL de tu cuenta Snowflake | `abc123.us-east-1` | ✅ |
| **SNOWFLAKE_USER** | Usuario de Snowflake | `mi_usuario` | ✅ |
| **SNOWFLAKE_PASSWORD** | Contraseña | `MiPass123!` | ✅ |
| **SNOWFLAKE_DATABASE** | Base de datos | `NY_TAXI` | ✅ |
| **SNOWFLAKE_WAREHOUSE** | Warehouse a usar | `COMPUTE_WH` | ✅ |
| **SNOWFLAKE_SCHEMA_RAW** | Schema de staging | `RAW` | ❌ (default: RAW) |
| **SNOWFLAKE_SCHEMA_ANALYTICS** | Schema de analytics | `ANALYTICS` | ❌ (default: ANALYTICS) |
| **SNOWFLAKE_ROLE** | Role de Snowflake | `ACCOUNTADMIN` | ❌ (default: ACCOUNTADMIN) |
| **PARQUET_BASE_URL** | URL base de datos | `https://...` | ❌ (tiene default) |
| **START_YEAR** | Año inicial | `2015` | ❌ (default: 2015) |
| **END_YEAR** | Año final | `2025` | ❌ (default: 2025) |

### ⚠️ Seguridad


---

## 🔄 Ejecución del Pipeline

### Orden de Ejecución de Notebooks

Ejecuta los notebooks en el siguiente orden:

#### 1️⃣ **01_ingesta_parquet_raw.ipynb**

**Propósito:** Descarga e ingesta de archivos Parquet a schema RAW

**Parámetros configurables:**
```python
START_YEAR = 2015
END_YEAR = 2025
SERVICES = ['yellow', 'green']
```

**Tiempo estimado:** 2-7 horas (depende de red y tamaño y servicio)

**Salida:**
- Tablas: `raw.yellow_trips_YYYY_MM`, `raw.green_trips_YYYY_MM`
- Metadatos: run_id, service_type, source_year, source_month, ingested_at_utc

**Validaciones:**
- Conteo de registros por archivo
- Verificación de columnas esperadas
- Log de errores de descarga

---

#### 2️⃣ **02_enriquecimiento_y_unificacion.ipynb**

**Propósito:** Enriquecer con Taxi Zones y unificar esquemas Yellow/Green

**Parámetros configurables:**
```python
SCHEMA_RAW = 'RAW'
```

**Tiempo estimado:** 30-60 minutos

**Transformaciones:**
- Join con `taxi_zone_lookup` para nombres de zonas/boroughs
- Normalización de catálogos (vendor, payment_type, rate_code)
- Unificación de columnas Yellow/Green

**Salida:**
- Tablas intermedias enriquecidas en RAW

---

#### 3️⃣ **03_construccion_obt.ipynb**

**Propósito:** Construir la One Big Table (OBT) en ANALYTICS

**Parámetros configurables:**
```python
SCHEMA_ANALYTICS = 'ANALYTICS'
RUN_ID = 'run_20250119_001'
REBUILD_MODE = 'full'  # o 'incremental'
```

**Tiempo estimado:** 1-2 horas

**Derivadas calculadas:**
```sql
trip_duration_min = (dropoff_datetime - pickup_datetime) / 60
avg_speed_mph = trip_distance / (trip_duration_min / 60)
tip_pct = (tip_amount / fare_amount) * 100
```

**Salida:**
- Tabla: `analytics.obt_trips`
- Grano: 1 fila = 1 viaje

**Verificación de idempotencia:**
- Re-ejecutar con mismo RUN_ID no duplica datos
- Estrategia: MERGE basado en clave natural

---

#### 4️⃣ **04_validaciones_y_exploracion.ipynb**

**Propósito:** Validar calidad de datos y exploración inicial

**Validaciones implementadas:**
- ✅ Nulos en columnas críticas (pickup_datetime, location_id)
- ✅ Rangos lógicos (distancia ≥ 0, duración > 0)
- ✅ Coherencia de fechas (dropoff > pickup)
- ✅ Montos positivos (fare_amount, total_amount)
- ✅ Consistencia de sumas (total_amount vs componentes)

**Tiempo estimado:** 15-30 minutos

**Reportes generados:**
- Conteo de nulos por columna
- Estadísticas descriptivas
- Outliers identificados
- Métricas de calidad

---

#### 5️⃣ **05_data_analysis.ipynb**

**Propósito:** Responder 20 preguntas de negocio

**Tiempo estimado:** 30-45 minutos

**Preguntas cubiertas:** Ver sección [Preguntas de Negocio](#-preguntas-de-negocio)

---

## 🗄️ Diseño de Esquemas

### Schema RAW (Staging Layer)

#### Propósito
Aterrizaje espejo del origen con metadatos mínimos de lineage.

#### Tablas

##### `raw.yellow_taxi`

| Columna | Tipo | Descripción | Origen |
|---------|------|-------------|--------|
| `VendorID` | INTEGER | Proveedor (1=Creative, 2=VeriFone) | Parquet |
| `tpep_pickup_datetime` | TIMESTAMP | Fecha/hora de pickup | Parquet |
| `tpep_dropoff_datetime` | TIMESTAMP | Fecha/hora de dropoff | Parquet |
| `passenger_count` | INTEGER | Número de pasajeros | Parquet |
| `trip_distance` | DOUBLE | Distancia en millas | Parquet |
| `RatecodeID` | INTEGER | Código de tarifa | Parquet |
| `store_and_fwd_flag` | STRING | Flag de almacenamiento | Parquet |
| `PULocationID` | INTEGER | ID zona de pickup | Parquet |
| `DOLocationID` | INTEGER | ID zona de dropoff | Parquet |
| `payment_type` | INTEGER | Tipo de pago | Parquet |
| `fare_amount` | DOUBLE | Tarifa base | Parquet |
| `extra` | DOUBLE | Extras (rush hour, overnight) | Parquet |
| `mta_tax` | DOUBLE | Impuesto MTA | Parquet |
| `tip_amount` | DOUBLE | Propina | Parquet |
| `tolls_amount` | DOUBLE | Peajes | Parquet |
| `improvement_surcharge` | DOUBLE | Recargo de mejora | Parquet |
| `total_amount` | DOUBLE | Monto total | Parquet |
| `congestion_surcharge` | DOUBLE | Recargo por congestión | Parquet |
| `airport_fee` | DOUBLE | Tarifa de aeropuerto | Parquet |
| **`run_id`** | STRING | ID de ejecución | **Generado** |
| **`service_type`** | STRING | 'yellow' | **Generado** |
| **`source_year`** | INTEGER | Año del archivo | **Generado** |
| **`source_month`** | INTEGER | Mes del archivo | **Generado** |
| **`ingested_at_utc`** | TIMESTAMP | Timestamp de ingesta | **Generado** |
| **`source_path`** | STRING | URL del Parquet | **Generado** |

##### `raw.green_taxi`

Misma estructura que Yellow, con diferencias:
- `lpep_pickup_datetime` / `lpep_dropoff_datetime` (en lugar de tpep)
- `trip_type` adicional (1=Street-hail, 2=Dispatch)
- `service_type` = 'green'

##### `raw.taxi_zone_lookup`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `LocationID` | INTEGER | ID único de zona |
| `Borough` | STRING | Manhattan, Brooklyn, Queens, Bronx, Staten Island, EWR |
| `Zone` | STRING | Nombre específico de la zona |
| `service_zone` | STRING | Yellow Zone, Boro Zone, Airports, etc. |

---

### Schema ENRICHED (ENRICHMENT Layer)

#### Tabla: `aenriched.unified_taxi_enriched`

### Schema ANALYTICS (Consumption Layer)

#### Tabla: `analytics.obt_trips`

**Grano:** 1 fila = 1 viaje

**Estrategia de desnormalización:** One Big Table (OBT) con todas las dimensiones embebidas

#### Columnas de la OBT (70+ columnas)

##### 📅 **Dimensiones de Tiempo** (10 columnas)

| Columna | Tipo | Descripción | Lógica |
|---------|------|-------------|--------|
| `pickup_datetime` | TIMESTAMP | Fecha/hora completa de pickup | Del RAW |
| `dropoff_datetime` | TIMESTAMP | Fecha/hora completa de dropoff | Del RAW |
| `pickup_date` | DATE | Solo fecha de pickup | `CAST(pickup_datetime AS DATE)` |
| `dropoff_date` | DATE | Solo fecha de dropoff | `CAST(dropoff_datetime AS DATE)` |

## Consideraciones finales

✅Docker Compose levanta Spark y Jupyter Notebook.
✅Todas las credenciales/parámetros provienen de variables de ambiente (.env).
✅Cobertura 2015–2025 (Yellow/Green) cargada en raw con matriz y conteos por lote.
✅analytics.obt_trips creada con columnas mínimas, derivadas y metadatos.
✅Idempotencia verificada reingestando al menos un mes.
✅Validaciones básicas documentadas (nulos, rangos, coherencia).
✅20 preguntas respondidas (texto) usando la OBT.
✅README claro: pasos, variables, esquema, decisiones, troubleshooting.