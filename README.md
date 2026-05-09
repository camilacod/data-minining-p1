# Proyecto 1 - Data Mining UTEC
### Procesamiento y Analisis de Datos Masivos sobre MovieLens 20M

Notebook autocontenido para Databricks que cubre las 5 partes del proyecto sobre el dataset MovieLens 20M.

---

## Integrantes

- Frisancho, Christian
- Valverde, Camila
- Zuloeta, Marcelo

## 1. Contenido

```
run-databricks/
├── proyecto1_DM_databricks.ipynb   Notebook principal (Partes I a V)
├── graphs/                          Exports PNG de las 14 visualizaciones
└── README.md                        Este archivo
```

El notebook esta organizado en 5 partes consecutivas:

| Parte | Contenido | Donde corre |
|---|---|---|
| I | Preprocesamiento + EDA (genero, ratings, sparsity) | Spark agrega, pandas grafica |
| II | Conteos distribuidos + Benchmark RDD vs DataFrame | Spark distribuido |
| III | Similitud, MinHash y LSH (manual sobre Opcion A) | Spark distribuido |
| IV | FP-Growth para reglas de asociacion | Spark MLlib + pandas en post |
| V | Discusion critica y aplicabilidad real | Markdown |

Cada cabecera de parte dentro del notebook contiene un bloque `> **Ejecucion:** ...` que indica explicitamente si la subtarea corre en Spark distribuido o en local, y por que. La justificacion completa esta en la celda 0 ("Mapa de ejecucion").

---

## 2. Dataset

MovieLens 20M (https://grouplens.org/datasets/movielens/20m/)

| Tabla | Filas | Tamano |
|---|---|---|
| ratings.csv | 20,000,263 | ~500 MB |
| movies.csv | 27,278 | ~1.4 MB |
| tags.csv | 465,564 | ~17 MB |
| links.csv | 27,278 | ~700 KB |
| genome-scores.csv | 11,709,768 | ~325 MB |

Solo se usan `ratings.csv` y `movies.csv` en este notebook.

---

## 3. Dependencias

Todas vienen preinstaladas en **Databricks Runtime 18.2 ML** (o superior):

| Paquete | Version usada |
|---|---|
| Spark | 4.1.0 |
| Scala | 2.13 |
| Python | 3.10+ |
| PySpark | viene con Spark 4.1.0 |
| pandas | viene con DBR ML |
| numpy | viene con DBR ML |
| matplotlib | viene con DBR ML |
| seaborn | viene con DBR ML |

No se requiere `pip install` adicional. El runtime ML incluye todas las dependencias preinstaladas.

---

## 4. Configuracion del cluster usada

Esta es la configuracion exacta sobre la que se ejecutaron las mediciones de tiempo y los resultados reportados en este proyecto (Azure Databricks):

| Parametro | Valor |
|---|---|
| Databricks Runtime | 18.2 ML (Scala 2.13, Spark 4.1.0) |
| Photon acceleration | activado |
| Tipo de worker | `Standard_EC8ads_v5` (Azure Confidential Compute, AMD EPYC, 64 GB RAM, 8 cores) |
| Driver | mismo tipo que worker |
| Autoscaling | habilitado, min `1`, max `4` workers |
| Terminate after inactivity | 80 minutos |
| Modo | Machine Learning |

**Notas sobre la eleccion del cluster:**

  * El benchmark de 17.4x del DataFrame sobre RDD (Parte II) se midio en este perfil.
  * Photon es relevante en este workload porque acelera agregaciones (`groupBy`, `count`, `array_intersect`) y ordenamiento (`orderBy`), que son las operaciones dominantes en las Partes I, II y III.
  * El tipo `EC8ads_v5` pertenece a la serie de Confidential Compute de Azure con procesadores AMD EPYC. Es valido para este workload pero no es estrictamente necesario; un worker general purpose equivalente (`Standard_D8ads_v5`, mismo tamano sin confidential compute) tendria rendimiento comparable a menor costo.
  * El rango de autoscaling 1 a 4 es razonable para un dataset de este tamano: el sistema arranca con 1 worker y solo escala si hay celdas pesadas en cola; en la practica raramente se observan mas de 2 workers activos al mismo tiempo durante el run completo.

---

## 5. Como cargar el dataset en Databricks

**Opcion A - Databricks Volumes (recomendado, Unity Catalog):**

1. Crear un volumen en el catalogo (UI: Catalog -> tu_catalog -> tu_schema -> Create -> Volume).
2. Subir los CSV via UI o via CLI:
   ```bash
   databricks fs cp ratings.csv dbfs:/Volumes/<catalog>/<schema>/movielens-20m/
   databricks fs cp movies.csv  dbfs:/Volumes/<catalog>/<schema>/movielens-20m/
   ```
3. Ajustar la celda de configuracion (celda 1) del notebook:
   ```python
   DATA_PATH = "/Volumes/<catalog>/<schema>/movielens-20m/"
   ```

**Opcion B - DBFS legacy:**

1. Subir los CSV a `dbfs:/FileStore/movielens-20m/`.
2. Apuntar `DATA_PATH = "/dbfs/FileStore/movielens-20m/"` en la celda de configuracion.

---

## 6. Ejecucion

1. Importar `proyecto1_DM_databricks.ipynb` al workspace de Databricks (Workspace -> Import -> File).
2. Adjuntar al cluster.
3. Ejecutar `Run All`.

Tiempo total esperado: **~20 minutos** en el cluster recomendado.

| Seccion | Tiempo aproximado |
|---|---|
| Setup + carga CSV | 30 s |
| Preprocesamiento (Parte I.1) | 1 min |
| EDA (Parte I.2) |6 min |
| Benchmark Parte II | 1 min (incluye los 31 s del RDD) |
| Mini-EDA + filtros Parte III.0-3.1 | 1 min |
| Jaccard exacto Parte III.3 | 3 min |
| MinHash Parte III.4 | 2.5 min (5 valores de K) |
| LSH Parte III.5 | 1 min (5 configuraciones de b, r) |
| FP-Growth Parte IV | 4 min |

---

## 7. Outputs

Los graficos se renderizan inline en el notebook. La carpeta `graphs/` contiene exports PNG de las 14 visualizaciones para uso en el reporte:

| Archivo | Parte | Contenido |
|---|---|---|
| cell13_graph1.png | I | Distribucion de ratings (absoluta + binarizada %) |
| cell15_graph2.png | I | Calificaciones por usuario (histograma + boxplot log) |
| cell17_graph3.png | I | Calificaciones por pelicula (histograma + boxplot log) |
| cell19_graph4.png | I | Pareto: top usuarios concentran X% de ratings |
| cell22_graph5.png | I | Generos: catalogo vs ratings (porcentajes relativos) |
| cell23_graph6.png | I | Heatmap de sparsity por deciles |
| cell25_graph7.png | II | Top 20 peliculas por numero de calificaciones |
| cell29_graph8.png | III | Mini-EDA: likes por pelicula y por usuario (log-log) |
| cell36_graph9.png | III | Distribucion de Jaccard exacto por estrato |
| cell40_graph10.png | III | MAE / RMSE / Pearson de MinHash vs K |
| cell40_graph11.png | III | Scatter Jaccard real vs estimado en K=128 |
| cell43_graph12.png | III | Precision / Recall / F1 vs (b, r) en LSH |
| cell43_graph13.png | III | Curva teorica `P(s) = 1 - (1 - s^r)^b` |
| cell51_graph14.png | IV | Reglas FP-Growth: soporte vs confianza + top 10 por lift |

---

## 8. Reproducibilidad

  * El notebook fija `random.seed(42)` y `np.random.seed(42)` antes de generar los hashes MinHash y de muestrear pares aleatorios. Las cifras del barrido de K, del barrido de (b, r) y de los top-N pares son deterministas y deberian reproducirse exactamente al ejecutar.
  * FP-Growth en MLlib es deterministico dado el mismo input y los mismos parametros (`minSupport`, `minConfidence`).
  * El benchmark RDD vs DataFrame mide tiempos absolutos que dependen del cluster; el speedup relativo (~17x) es estable a traves de configuraciones razonables.

---

## 9. Decisiones de implementacion

  * **Representacion (Parte III):** Opcion A (cada pelicula es el conjunto de usuarios que le dieron like). Justificacion completa en la celda 30 del notebook.
  * **Umbral like / dislike:** rating >= 3.5. Produce un balance ~61% likes / 39% dislikes, suficiente para dejar senal en ambas clases.
  * **Cortes de filtrado en Parte III:** 50 likes minimos por pelicula, 20 likes minimos por usuario. Justificados por el mini-EDA de la celda 32.
  * **K = 128 hashes para LSH:** elegido tras barrer K en {16, 32, 64, 128, 256}. Mejor compromiso entre precision (Pearson 0.88) y costo (128 enteros por pelicula).
  * **Barrido (b, r):** 5 configuraciones con `b * r = 128`: (8, 16), (16, 8), (32, 4), (64, 2), (128, 1).
  * **FP-Growth:** `minSupport = 0.05` (>= 5,350 usuarios), `minConfidence = 0.5`. Genera 2.2M reglas con lift hasta 9.22.

---

## 10. Estructura del notebook (61 celdas)

```
[ 0] Header + Mapa de ejecucion Spark/Local
[ 1] Setup (SparkSession, paths, imports)
[ 2-26]  PARTE I    Preprocesamiento y EDA
[27-29]  PARTE II   Procesamiento distribuido + Benchmark
[30-49]  PARTE III  Similitud, MinHash y LSH
[50-59]  PARTE IV   FP-Growth y reglas
[   60]  PARTE V    Discusion y reflexion
```

Cada parte mayor abre con un bloque `> **Ejecucion:** Spark / Local / por que` que cumple el requerimiento del enunciado de indicar explicitamente donde corre cada modulo.

---

## 11. Referencias

  * MovieLens 20M Dataset: F. M. Harper and J. A. Konstan. The MovieLens Datasets: History and Context. ACM TiiS, 2015.
  * MinHash: Broder, A. Z. (1997). On the Resemblance and Containment of Documents.
  * LSH: Indyk, P. and Motwani, R. (1998). Approximate Nearest Neighbors: Towards Removing the Curse of Dimensionality.
  * FP-Growth: Han, J., Pei, J., and Yin, Y. (2000). Mining Frequent Patterns without Candidate Generation.
  * Spark optimizations: Armbrust et al. (2015). Spark SQL: Relational Data Processing in Spark.
