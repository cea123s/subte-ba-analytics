# Documentación Técnica
**Subtes de Buenos Aires (2013-2026)**

## 1. Origen y Fuentes de Datos
El proyecto integra la serie histórica pública provista por el Gobierno de la Ciudad Autónoma de Buenos Aires (GCBA) a través del portal [Datos Abiertos del GCBA](https://data.buenosaires.gob.ar/dataset/subte-viajes-molinetes "Ir a la página de BA Data"):

*   **Proveedor de datos:** Datos Abiertos GCBA - Subterráneos de Buenos Aires S.E. (SBASE).
*   **Ventana temporal analizada:** Junio de 2013 a Junio de 2026 (últimos datos disponibles a la fecha de análisis).
*   **Granularidad original:** Registros de molinetes agrupados en intervalos de 15 minutos.
*   **Segmentación de lotes de entrada:**
    *   **Conjunto Histórico 1 (2013-2021):** Archivos CSV anuales y mensuales con codificación mixta (predominancia de UTF-8 y Latin-1) y formatos de fecha combinados.
    *   **Conjunto Reciente 2 (2022-2026):** Archivos CSV particionados por grupos de líneas (ABC, DEH, PM) con encapsulado completo entre comillas, separador de campos por punto y coma (`;`) y prefijo BOM UTF-8 (`ï»¿`).
*   **Shape dataset final:** 145.810.220 filas x 10 columnas.

---

## 2. Pipeline de Extracción, Transformación y Carga (ETL en Python)¹

Se implementó un pipeline en Python con Pandas para resolver anomalías estructurales y garantizar la coherencia a lo largo de todo el dataset.

### 2.1. Resolución de Ambigüedad en Fechas
*   **Falla original:** Los lotes 2013-2021 contenían fechas ISO (`AAAA-MM-DD`), mientras que el lote 2022-2026 incorporó la convención local `D/M/AAAA` (ej. `12/6/2026`). El motor de parseo automático con `dayfirst=True` malinterpretaba valores como `2026-06-12` convirtiéndolos en `2026-12-06`, extendiendo erróneamente el calendario hasta diciembre de 2026.
*   **Solución:** Se programó una función de "parseo en cascada". Las fechas pasan por 4 tipos de parseos posibles: aquellos que fallan son completados por otros.

### 2.2. Normalización de Esquemas y Metadatos
*   **Encapsulado y BOM:** Lectura cruda con `quoting=3` para interpretación literal de comillas perimetrales y eliminación del caracter `BOM ï»¿` en encabezados.
*   **Desglose de infraestructura:** Se extrajo la boca y número de molinete mediante REGEX sobre el identificador del molinete y se aisló la letra de la línea (A, B, C, D, E, H, M).
*   **Franjas horarias:** Homogeneización de horas `HH:MM:SS` y `HH:MM` extrayendo la hora entera como variable numérica discreta (`HORA`, valores de 0 a 23 `Int64`).

### 2.3. Optimización de RAM, espacio en disco y procesamiento
*   **Problema:** Se tardaba muchísimo tiempo en procesar grandes cantidades de datos al cometer dos errores: trabajar directamente sobre archivos `.csv` y trabajar de a lotes concatenados enormes. La memoria RAM se colapsaba y los tiempos de procesamiento eran muy grandes.
*   **Solución:** Pasar rápidamente los archivos procesados a formato `.parquet` y mantener una estructura atomizada en años. Esto redujo inmediatamente el consumo de RAM, la intensidad y tiempos del procesamiento (¡los cooler de la computadora dejaron de dispararse como locos!).

> *¹ Más detalle y análisis en profundidad en la notebook de implementación del pipeline ETL.*

---

## 3. Arquitectura del Modelo de Datos en Power BI

El diseño relacional responde a un esquema en estrella.

| Tabla | Tipo | Columnas / Atributos | Descripción |
| :--- | :--- | :--- | :--- |
| **fact_pasajeros** | Hechos (Fact) | `FECHA`, `DESDE`, `HASTA`, `HORA`, `LINEA`, `ESTACION`, `BOCA`, `MOLINETE`, `PAX_TOTAL` | Tabla combinada consolidada. |
| **dim_calendario** | Dimensión (Dimension) | `Date`, `Anio`, `Mes`, `MesNro`, `MesAnio`, `DiaSemana`, `EsFinDeSemana` | Generada por DAX dinámico. Nombres forzados con configuración regional "es-AR" y ordenados por su índice numérico. |
| **_Medidas** | Contenedor DAX | Medidas de cálculo. | Tabla dedicada a centralizar los cálculos del negocio. |

---

## 4. Diccionario de Medidas DAX

### 4.1. Volumen y Métricas Base
Este grupo agrupa los cálculos absolutos que sirven como cimientos para el resto del modelo.
*   **Total_Pasajeros:** Calcula la suma absoluta del volumen de usuarios que ingresaron al sistema.
*   **Promedio_Diario_PAX:** Media de pasajeros diarios para filtrar fluctuaciones atípicas y permitir comparaciones más equilibradas.
*   **Molinetes_Activos:** Contabiliza la cantidad de molinetes operativos para dimensionar la capacidad instalada.

### 4.2. Análisis Relativo y Participación
Medidas destinadas a entender proporciones dentro de la red.
*   **%_del_Total_PAX:** Determina la cuota de participación porcentual de una entidad o segmento sobre el volumen total de la red.
*   **PAX_Por_Molinete:** Evalúa la intensidad y el estrés operativo dividiendo el volumen de usuarios por la cantidad de molinetes activos.

### 4.3. Identificación de Picos y Extremos
Fórmulas dinámicas para extraer texto o valores máximos que alimentan las tarjetas informativas (KPIs).
*   **Boca_Mayor_Demanda:** Devuelve el nombre textual de la boca específica que registró el mayor caudal de usuarios en el contexto seleccionado.
*   **Dia_Mayor_Demanda:** Identifica textualmente el día de la semana con mayor concentración de pasajeros.
*   **Estacion_Mayor_Demanda:** Señala el nombre de la estación de la red con el máximo volumen de pasajeros.
*   **Hora_Pico_Mayor_Demanda:** Detecta la ventana horaria donde se produce el pico de mayor demanda.

### 4.4 Tiempo y Variaciones
Cálculos para medir el rendimiento histórico y la recuperación del servicio.
*   **PAX_Base_2019:** Aísla el volumen de pasajeros del año 2019 para usarlo como línea base pre-pandemia.
*   **Recuperacion_vs_2019_%:** Mide porcentualmente qué tan cerca está el volumen actual respecto a los niveles del 2019.
*   **Variacion_YoY_%:** Calcula la variación porcentual interanual (Year-over-Year) contrastando el período actual contra el mismo del año anterior.

### 4.5 Control Visual
Medidas auxiliares creadas exclusivamente para facilitar el análisis visual y mejorar la interacción del usuario con el tablero.
*   **Color_Linea_Subte:** Asigna dinámicamente el código hexadecimal oficial de cada ramal para unificar la identidad visual de los gráficos.
*   **Recu.2019_Tarjeta:** Cuando no hay ningún año seleccionado, en vez de quedar vacía la tarjeta, se muestra un texto orientativo ("Seleccione un año").
*   **YoY_Tarjeta:** Cuando no hay ningún año seleccionado, en vez de quedar vacía la tarjeta, se muestra un texto orientativo ("Seleccione un año").

---

## 5. Mantenimiento Futuro

1.  **Cierre de Serie Temporal:** Los registros finalizan el 30 de junio de 2026. Al comparar el año 2026 contra ejercicios precedentes en dashboards anuales, debe priorizarse siempre la métrica `[Promedio_Diario_PAX]` para evitar lecturas distorsionadas de caída de volumen.
2.  **Ingesta de Lotes Futuros:** Todo nuevo lote mensual provisto por GCBA debe procesarse con el script de parseo posicional antes de su anexión, validando que no se generen registros vacíos en `FECHA` ni valores huérfanos frente a `dim_calendario`. Mientras se siga respetando la codificación del segundo conjunto, se podrá aplicar el script correspondiente al período 2022-2026.
3.  **Consistencia Categórica:** En Power BI, las columnas textuales (`Mes` y `DiaSemana`) deben mantener asignada su clave de ordenamiento utilizando `MesNro` y `DiaSemanaNro`.
4.  **Informe Ejecutivo:** Para la elaboración de dicho informe ejecutivo y embeberse en la terminología específica de los subterráneos, se tomaron como referencia algunos de los informes de [Indicadores y balances de Buenos Aires Ciudad](https://buenosaires.gob.ar/gcaba_historico/indicadores-y-balances).