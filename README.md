# Proyecto de Análisis de Datos del S&P 500

&#x20;&#x20;

## Descripción

Repositorio con el pipeline completo de ingestión, limpieza, modelado y visualización de datos históricos de las 502 compañías que conforman el S&P 500. El objetivo es demostrar habilidades en ingeniería de datos, BI y documentación profesional para un portafolio atractivo ante potenciales empleadores.

## Tabla de Contenidos

- [Estructura del Repositorio](#estructura-del-repositorio)
- [Arquitectura de Datos (Medallion)](#arquitectura-de-datos-medallion)
  - [Capa Bronze](#capa-bronze)
  - [Capa Silver](#capa-silver)
  - [Capa Gold](#capa-gold)
- [Origen de Datos](#origen-de-datos)
- [Requisitos Previos](#requisitos-previos)
- [Configuración del Entorno](#configuración-del-entorno)
- [Pipeline de Datos](#pipeline-de-datos)
  - [Carga en Bronze](#carga-en-bronze)
  - [Limpieza y Filtrado (Silver)](#limpieza-y-filtrado-silver)
  - [Modelado Dimensional y Clustering (Gold)](#modelado-dimensional-y-clustering-gold)
- [Script de Recarga Automática](#script-de-recarga-automática)
- [Conectar Power BI](#conectar-power-bi)
- [Datos de Ejemplo](#datos-de-ejemplo)
- [Buenas Prácticas](#buenas-prácticas)
- [Licencia](#licencia)
- [Contacto](#contacto)

## Estructura del Repositorio

```text
sp500-analysis/
├── data/
│   ├── csv/               ← 502 archivos .csv originales
│   └── sample/            ← datos de ejemplo para pruebas rápidas
├── scripts/               ← scripts de recarga y utilidades
│   └── reload_data.sh     ← recarga completa en BigQuery Sandbox
├── sql/
│   ├── bronze/            ← DDL y consultas Bronze
│   ├── silver/            ← transformaciones Silver
│   └── gold/              ← creación de dimensiones y hechos Gold
├── powerbi/               ← archivos .pbix y capturas de pantalla
├── docs/                  ← mockups y diagramas (Canva/Figma)
├── README.md              ← documentación principal
└── LICENSE                ← licencia del proyecto
```

## Arquitectura de Datos (Medallion)

### Capa Bronze

- **Tabla**: `sp500_bronze.raw_all`
- **Descripción**: datos crudos, sin transformar, cargados desde los 502 CSV.
- **Partición**: N/A (opcional clustering en Gold).

### Capa Silver

- **Tabla**: `sp500_bronze.prices_filtered`
- **Descripción**: fechas y valores filtrados desde `2000-01-01` y cambio de nombres a columnas en minúsculas.
- **Dimensión de empresas**: `sp500_silver.dim_companies`.

### Capa Gold

- **Dimensiones**:
  - `sp500_gold.dim_date` (año, mes, día, trimestre)
  - `sp500_gold.dim_company` (company\_id, ticker, sector, industria)
- **Hechos**:
  - `sp500_gold.fact_prices`
- **Optimización**: clustering por `fecha, ticker` para acelerar consultas.

## Origen de Datos

- **502 archivos CSV** con columnas: `Date,Open,High,Low,Close,Adj Close,Volume`.
- **Archivo de categorías**: `categories.csv` con `ticker,sector,industry`.

## Requisitos Previos

- Cuenta de Google Cloud con **BigQuery Sandbox** habilitado.
- **Cloud Shell** para ejecutar comandos `bq` y `gcloud`.
- Power BI Desktop para visualizar los dashboards.

## Configuración del Entorno

1. Clonar el repositorio:
   ```bash
   git clone https://github.com/tu-usuario/sp500-analysis.git
   cd sp500-analysis
   ```
2. Configurar proyecto en Cloud Shell:
   ```bash
   gcloud auth login
   gcloud config set project <TU_PROJECT_ID>
   ```
3. Crear datasets en BigQuery Sandbox:
   ```bash
   bq --location=US mk --dataset sp500_bronze
   bq --location=US mk --dataset sp500_silver
   bq --location=US mk --dataset sp500_gold
   ```

## Pipeline de Datos

### Carga en Bronze

Los datos se cargan en `sp500_bronze.raw_all` con el script:

```bash
bash scripts/reload_data.sh
```

Este script:

1. Crea o limpia el dataset (`sp500_bronze`).
2. Carga cada CSV añadiéndolo con `bq load --noreplace`.

### Limpieza y Filtrado (Silver)

```sql
CREATE OR REPLACE TABLE `sp500_bronze.prices_filtered` AS
SELECT
  Date   AS fecha,
  Open   AS open,
  High   AS high,
  Low    AS low,
  Close  AS close,
  `Adj Close` AS adj_close,
  Volume AS volume
FROM `sp500_bronze.raw_all`
WHERE Date >= "2000-01-01";
```

### Modelado Dimensional y Clustering (Gold)

```sql
CREATE OR REPLACE TABLE `sp500_gold.dim_date` AS
SELECT
  fecha,
  EXTRACT(YEAR FROM fecha)   AS año,
  EXTRACT(MONTH FROM fecha)  AS mes,
  EXTRACT(DAY FROM fecha)    AS día,
  EXTRACT(QUARTER FROM fecha) AS trimestre
FROM UNNEST(GENERATE_DATE_ARRAY("2000-01-01", CURRENT_DATE(), INTERVAL 1 DAY)) AS fecha;

CREATE OR REPLACE TABLE `sp500_gold.dim_company` AS
SELECT
  ROW_NUMBER() OVER(ORDER BY ticker) AS company_id,
  ticker,
  sector,
  industry
FROM `sp500_silver.dim_companies`;

CREATE OR REPLACE TABLE `sp500_gold.fact_prices`
CLUSTER BY fecha, ticker AS
SELECT
  c.company_id,
  p.fecha,
  p.open,
  p.high,
  p.low,
  p.close,
  p.adj_close,
  p.volume,
  p.ticker
FROM `sp500_bronze.prices_filtered` p
JOIN `sp500_gold.dim_company` c USING(ticker);
```

## Script de Recarga Automática

En `scripts/reload_data.sh` encontrarás el flujo completo para reconstruir la capa Bronze en BigQuery Sandbox. Simplemente coloca tus 502 CSV en `data/csv/` y ejecuta:

```bash
bash scripts/reload_data.sh
```

## Conectar Power BI

1. En Power BI Desktop, elige **Obtener datos → Google BigQuery**.
2. Conéctate a tu proyecto y selecciona `sp500_gold.dim_date`, `sp500_gold.dim_company` y `sp500_gold.fact_prices`.
3. Configura relaciones:
   - fact\_prices[fecha] ↔ dim\_date[fecha]
   - fact\_prices[company\_id] ↔ dim\_company[company\_id]
4. Diseña tu dashboard con tarjetas, gráficos de líneas y segmentaciones.

## Datos de Ejemplo

Para pruebas rápidas sin los 502 CSV completos, en `data/sample/` hay un subconjunto de compañías. Ejecuta igual el script de recarga apuntando a esa carpeta para ver el pipeline en minutos.

## Buenas Prácticas

- Commits atómicos y mensajes descriptivos.
- Branching: usa `main`, `dev`, `feature/...`.
- Documenta cada script y consulta SQL.
- Usa GitHub Actions (en `.github/workflows/`) para validar sintaxis SQL.

## Licencia

Este proyecto está bajo la licencia MIT. Consulta el archivo [LICENSE](LICENSE) para más detalles.

## Contacto

Marcos y Nico – [tu-email@ejemplo.com](mailto\:tu-email@ejemplo.com)\
LinkedIn: [https://www.linkedin.com/in/tu-perfil](https://www.linkedin.com/in/tu-perfil)

