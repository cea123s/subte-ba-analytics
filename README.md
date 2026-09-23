# Análisis y Modelado de Datos: Red de Subtes de Buenos Aires (2013-2026)

## Descripción del Proyecto
Análisis *end-to-end* del uso e infraestructura operativa de la red de subterráneos de la Ciudad de Buenos Aires. El proyecto procesa más de **145 millones de registros históricos** aplicando un *pipeline* ETL para normalizar inconsistencias, optimizar los recursos de _hardware_ y estructurar un modelo analítico en *Power BI*. 

El objetivo analítico evalúa la demanda estructural post-pandemia, identifica cuellos de botella en la infraestructura de validación y propone **estrategias operativas basadas en datos**.

El proyecto entrega:

* [Informe ejecutivo](<docs/Informe Ejecutivo.pdf>) con el analisis completo.
* [Documentación Técnica](docs/documentacion_tecnica.md) en dónde se detallan los aspectos clave del proceso.
* [Tablero en Power BI](imgs/) que permite profundizar en múltiples dimensiones.
* [Notebooks](scripts/etl_unificado.ipynb) con todo el proceso ETL documentado al detalle.

---

## Key Insights (Foco Operativo 2022-2025)
* **Contracción Estructural:** El techo de demanda se redujo un 30% de forma permanente frente a valores pre-COVID, estabilizándose en ~668.000 pasajeros diarios (2023).
* **Redistribución de Tráfico:** Las líneas B y D perdieron 7,1 puntos de participación conjunta. La Línea H incrementó su volumen (+69% relativo), consolidando su rol de anillo transversal.
* **Estrés de Infraestructura:** La Línea B opera con un riesgo crítico (19,4 M pasajeros/molinete), exigiendo su infraestructura 5 veces más que la Línea C (3,7 M). Estaciones como Federico Lacroze, J.M. de Rosas y Congreso de Tucumán exhiben asimetrías y cuellos de botella severos en sus accesos.

---

## Tech Stack
* **Lenguaje:** Python
* **Procesamiento de Datos:** pandas, charset_normalizer, os, pyspark
* **Visualización y Modelado:** Power BI, DAX
* **Formatos de Almacenamiento:** CSV, Parquet

## Arquitectura y Pipeline ETL
El procesamiento se dividió en dos lotes temporales debido a cambios estructurales severos en los datasets crudos (formatos de fecha, *encodings* mixtos, *quoting* encapsulado, BOM UTF-8).
* **Parseo en Cascada:** Implementación de funciones secuenciales para resolver ambigüedad en formatos de fecha (ISO 8601 vs D/M/AAAA).
* **Detección de Encoding:** Algoritmo de sampleo y *fallback* dinámico (UTF-8, ASCII, CP1250, Latin-1).
* **Extracción de Metadatos:** Aislamiento de infraestructura operativa (Línea, Boca y Molinete) mediante *regex* sobre cadenas crudas.
* **Optimización de Rendimiento:** Conversión transaccional inmediata de archivos procesados a formato `.parquet` comprimido (Snappy), reduciendo drásticamente el consumo de RAM.

## Data Model (Star Schema)
Modelo relacional implementado en Power BI:
* `fact_pasajeros`: 145.810.220 registros atómicos con FECHA, HORA, LINEA, ESTACION, BOCA, MOLINETE y PAX_TOTAL
* `dim_calendario`: Dimensión de tiempo generada dinámicamente vía DAX.
* `_Medidas`: Contenedor centralizado de métricas de negocio (*Promedio_Diario_PAX*, *PAX_Por_Molinete*, *Recuperacion_vs_2019_%*, etc).

## Instrucciones de Reproducción
1. Clonar el repositorio.
2. Descargar la serie completa de datasets de la página de [Datos Abiertos del GCBA](https://data.buenosaires.gob.ar/dataset/subte-viajes-molinetes "Ir a la página de BA Data").
3. Ordenar los datasets en sus carpetas correspondientes: 2013-2021 y 2022-2026.
4. Renombrar los datasets del primer periodo como: _molinetes_20**.csv_ (ej. molinetes_2019.csv)
5. Organizar los datasets del segundo periodo por carpetetas (ej. molinetes-2024) y volver adentro los dataset correspondientes.
6. Ejecutar `etl_unificado.ipynb` para generar los archivos `.parquet` consolidados en la carpeta `/scripts/output/`.
7. Abrir `tablero_subtes.pbix` actualizando el origen de datos al directorio local.

---
## Estructura del repositorio

```
subtes-ba-analytics/
├── docs/                                  # Documentación funcional y técnica
│   ├── Informe Ejecutivo.pdf              # Diagnóstico operativo y resiliencia de la red
│   └── documentacion_tecnica.md           # Arquitectura del modelo, pipeline ETL y diccionario DAX
├── scripts/                               # Código fuente y pipelines
│   ├── eda/                               # Análisis Exploratorio de Datos
│   │   ├── eda_2020.ipynb                 # Exploración inicial del dataset crudo (2020)
│   │   ├── eda_13-21.ipynb                # Verificación de integridad del período 2013-2021 (post etl)
│   │   └── BaseUnificadaEstaciones.xlsx   # Dataset base para EDA 2020
│   └── etl_unificado.ipynb                # Pipeline ETL principal (extracción, transformación y carga)
├── imgs/                                  # Capturas del tablero en Power BI
├── .gitignore
├── requirements.txt
└── README.md
```