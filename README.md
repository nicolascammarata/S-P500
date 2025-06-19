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

# Proyecto de Análisis de Datos del S\&P 500

&#x20;&#x20;

## Descripción

Repositorio con el pipeline completo de ingestión, limpieza, modelado y visualización de datos históricos de las 502 compañías que conforman el S\&P 500. El objetivo es demostrar habilidades en ingeniería de datos, BI y documentación profesional para un portafolio atractivo ante potenciales empleadores.

## Tabla de Contenidos

* [Estructura del Repositorio](#estructura-del-repositorio)
* [Arquitectura de Datos (Medallion)](#arquitectura-de-datos-medallion)

  * [Capa Bronze](#capa-bronze)
  * [Capa Silver](#capa-silver)
  * [Capa Gold](#capa-gold)
* [Origen de Datos](#origen-de-datos)
* [Requisitos Previos](#requisitos-previos)
* [Configuración del Entorno](#configuración-del-entorno)
* [Pipeline de Datos](#pipeline-de-datos)

  * [Carga en Bronze](#carga-en-bronze)
  * [Limpieza y Filtrado (Silver)](#limpieza-y-filtrado-silver)
  * [Modelado Dimensional y Clustering (Gold)](#modelado-dimensional-y-clustering-gold)
* [Script de Recarga Automática](#script-de-recarga-automática)
* [Conectar Power BI](#conectar-power-bi)
* [Datos de Ejemplo](#datos-de-ejemplo)
* [Buenas Prácticas](#buenas-prácticas)
* [Licencia](#licencia)
* [Contacto](#contacto)

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

El modelo Medallion organiza el pipeline por capas de creciente calidad y preparación para el análisis:

### Capa Bronze

* **Propósito**: Ingestión y preprocesado mínimo. Mantiene la fidelidad a los archivos originales pero ya con filtros básicos y audit metadata.
* **Acciones**:

  1. Cargar los 502 CSV en `sp500_bronze.raw_all`.
  2. Filtrar registros anteriores al **2002-01-01** (rango base).
  3. Renombrar columnas a nombres estandarizados (`Date`→`fecha`, etc.).
  4. Añadir columna `ticker` extraída del nombre de archivo.
  5. Aplicar **clustering** por `fecha` y `ticker` para optimizar lecturas.
* **Tabla resultante**: `sp500_bronze.prices_bronze`

### Capa Silver

* **Propósito**: Limpieza adicional y enriquecimiento con dimensiones de referencia.
* **Acciones**:

  1. Crear `sp500_silver.prices_filtered` a partir de `prices_bronze`, conservando sólo columnas relevantes y versiones tipadas (DATE, NUMERIC).
  2. Unir con `categories.csv` para generar la dimensión de empresas:

     ```sql
     CREATE OR REPLACE TABLE sp500_silver.dim_companies AS
     SELECT DISTINCT ticker, sector, industry
     FROM sp500_silver.categories_raw
     WHERE ticker IN (SELECT DISTINCT ticker FROM sp500_silver.prices_filtered);
     ```
* **Tablas resultantes**:

  * `sp500_silver.prices_filtered`
  * `sp500_silver.dim_companies`

### Capa Gold

* **Propósito**: Modelado dimensional listo para análisis y visualización.
* **Acciones**:

  1. Generar dimensión de fechas `sp500_gold.dim_date` (año, mes, día, trimestre).
  2. Crear `sp500_gold.dim_company` con IDs únicos para cada ticker.
  3. Construir la tabla de hechos `sp500_gold.fact_prices`, juntando hechos de precio con dimensiones:

     ```sql
     CREATE OR REPLACE TABLE sp500_gold.fact_prices
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
     FROM sp500_silver.prices_filtered p
     JOIN sp500_gold.dim_company c USING(ticker);
     ```
* **Tabla resultante**: `sp500_gold.fact_prices`

## Origen de Datos

* **502 archivos CSV** con columnas: `Date,Open,High,Low,Close,Adj Close,Volume`.
* **Archivo de categorías**: `categories.csv` con `ticker,sector,industry`.

## Requisitos Previos

* Cuenta de Google Cloud con **BigQuery Sandbox** habilitado.
* **Cloud Shell** para ejecutar comandos `bq` y `gcloud`.
* Power BI Desktop para visualizar los dashboards.

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
  Volume AS volume,
  -- Extrae ticker del nombre de archivo
  REGEXP_EXTRACT(_FILE_NAME(), r'([^/]+)\.csv$') AS ticker
FROM `sp500_bronze.raw_all`
WHERE Date >= "2002-01-01";
```

1. **Join con categorías**:

   ```sql
   CREATE OR REPLACE TABLE `sp500_silver.prices_enriched` AS
   SELECT
     p.*, c.sector, c.industry
   FROM `sp500_bronze.prices_filtered` p
   LEFT JOIN `sp500_silver.categories_raw` c
     USING(ticker);
   ```
2. **Dimensión de empresas**:

   ```sql
   CREATE OR REPLACE TABLE sp500_silver.dim_companies AS
   SELECT DISTINCT ticker, sector, industry
   FROM sp500_silver.prices_enriched;
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

## Análisis de Variación

En esta sección se muestran ejemplos de consultas para calcular la **variación porcentual** de las compañías entre dos fechas y *rankear* las que más han crecido o caído en un periodo determinado.

### Cálculo de Porcentajes de Cambio

```sql
-- Porcentaje de cambio de precio de cierre entre 2020-01-01 y 2021-12-31
WITH base AS (
  SELECT
    ticker,
    FIRST_VALUE(close) OVER (
      PARTITION BY ticker
      ORDER BY fecha
      RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS precio_inicio,
    LAST_VALUE(close) OVER (
      PARTITION BY ticker
      ORDER BY fecha
      RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS precio_fin
  FROM `animated-memory-463400-r9.sp500_silver.prices_enriched`
  WHERE fecha BETWEEN '2020-01-01' AND '2021-12-31'
)
SELECT
  ticker,
  SAFE_DIVIDE(precio_fin - precio_inicio, precio_inicio) * 100 AS porcentaje_cambio
FROM base;
```

### Ranking de Variación

```sql
-- Top 10 empresas con mayor crecimiento porcentual en 2020–2021
SELECT
  ticker,
  porcentaje_cambio
FROM (
  -- reutiliza la consulta anterior como subconsulta
  WITH base AS (
    SELECT
      ticker,
      FIRST_VALUE(close) OVER (
        PARTITION BY ticker
        ORDER BY fecha
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
      ) AS precio_inicio,
      LAST_VALUE(close) OVER (
        PARTITION BY ticker
        ORDER BY fecha
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
      ) AS precio_fin
    FROM `animated-memory-463400-r9.sp500_silver.prices_enriched`
    WHERE fecha BETWEEN '2020-01-01' AND '2021-12-31'
  )
  SELECT
    ticker,
    SAFE_DIVIDE(precio_fin - precio_inicio, precio_inicio) * 100 AS porcentaje_cambio
  FROM base
)
ORDER BY porcentaje_cambio DESC
LIMIT 10;
```

> Ajusta los rangos de fecha en `WHERE` para analizar otros periodos.

## Script de Recarga Automática

En `scripts/reload_data.sh` encontrarás el flujo completo para reconstruir la capa Bronze en BigQuery Sandbox. Simplemente coloca tus 502 CSV en `data/csv/` y ejecuta:

```bash
bash scripts/reload_data.sh
```

## Conectar Power BI

1. En Power BI Desktop, elige **Obtener datos → Google BigQuery**.
2. Conéctate a tu proyecto y selecciona `sp500_gold.dim_date`, `sp500_gold.dim_company` y `sp500_gold.fact_prices`.
3. Configura relaciones:

   * fact\_prices\[fecha] ↔ dim\_date\[fecha]
   * fact\_prices\[company\_id] ↔ dim\_company\[company\_id]
4. Diseña tu dashboard con tarjetas, gráficos de líneas y segmentaciones.

## Datos de Ejemplo

Para pruebas rápidas sin los 502 CSV completos, en `data/sample/` hay un subconjunto de compañías. Ejecuta igual el script de recarga apuntando a esa carpeta para ver el pipeline en minutos.

## Buenas Prácticas

* Commits atómicos y mensajes descriptivos.
* Branching: usa `main`, `dev`, `feature/...`.
* Documenta cada script y consulta SQL.
* Usa GitHub Actions (en `.github/workflows/`) para validar sintaxis SQL.

## Licencia

Este proyecto está bajo la licencia MIT. Consulta el archivo [LICENSE](LICENSE) para más detalles.

## Contacto

Marcos y Nico – [tu-email@ejemplo.com](mailto:tu-email@ejemplo.com)
LinkedIn: [https://www.linkedin.com/in/tu-perfil](https://www.linkedin.com/in/tu-perfil)
