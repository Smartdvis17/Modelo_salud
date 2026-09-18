# Modelo Salud — Clustering de perfiles de riesgo de intento de suicidio

Proyecto de aprendizaje no supervisado que segmenta los casos de intento de suicidio del municipio de Tunja, Boyacá, en perfiles de riesgo mediante **MCA (Análisis de Correspondencias Múltiples) + K-Means**.

## Fuente de datos

- **Dataset**: Información Intentos de Suicidio, Municipio de Tunja, Boyacá.
- **Origen**: [datos.gov.co](https://www.datos.gov.co/Salud-y-Protecci-n-Social/Informaci-n-Intentos-de-Suicidio-Municipio-de-Tunj/nk8x-s9hw/about_data) (dato público).
- **Tamaño**: 818 registros, 48 columnas originales (2020 – marzo 2024).

> **Nota de manejo responsable de datos**: es información sensible de salud mental. El dataset es público y no incluye identificadores directos (nombre, documento), pero sí variables cuasi-identificadoras (barrio de residencia, fecha, edad). Este repositorio se usa con fines de portafolio; cualquier uso derivado para decisiones reales de política pública debe pasar por un comité de ética y anonimización adicional.

## Licencia y uso de datos

**Dataset**: licencia [Creative Commons Atribución-CompartirIgual 4.0 Internacional (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/legalcode), atribución **Alcaldía de Tunja, Boyacá** (verificado en el metadato oficial del dataset en datos.gov.co). Este repositorio no redistribuye el dataset (ver `.gitignore`); solo lo referencia y lo transforma para producir el análisis publicado aquí.

**Código, notebooks y análisis propios**: © Ana Salcedo Martínez, licencia [Creative Commons Atribución-NoComercial-CompartirIgual 4.0 (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode) — ver texto completo en [LICENSE](LICENSE). Cualquiera puede ver, compartir y adaptar este trabajo dando crédito, siempre que sea sin fines comerciales y que cualquier versión adaptada se comparta bajo esta misma licencia.

**Aviso de responsabilidad**: este es un proyecto académico/exploratorio, sin garantía de ningún tipo. No es una herramienta de diagnóstico ni de decisión clínica, no debe usarse para identificar o etiquetar individuos, y no se realizó ni se autoriza ningún intento de reidentificación de personas a partir de las variables cuasi-identificadoras del dataset. Cualquier aplicación en un contexto real de salud pública debe pasar primero por un comité de ética y por asesoría legal calificada (ej. cumplimiento de la Ley 1581 de 2012 de protección de datos personales en Colombia, que clasifica los datos de salud como datos sensibles).

## Estructura del proyecto

```
├── 01_Limpieza.ipynb                  # Carga, normalización de texto y fechas, imputación, categorización
├── 02_Analisis_desciptivo.ipynb       # EDA univariado/multivariado, correlaciones, hallazgos generales
├── 03_Preparacion_Clustering.ipynb    # Separación variables modelo / metadata / validación
├── 04_Clustering.ipynb                # MCA + K-Means, selección de k, estabilidad, perfilado y persistencia
├── data/
│   ├── data.csv                       # Dataset crudo
│   ├── data_clean.csv                 # Salida de 01
│   ├── data_cluster.csv               # Variables de entrada al modelo (salida de 03)
│   └── data_validacion.csv            # Metadata + variable de validación (salida de 03)
├── modelos/
│   ├── scaler/                        # mca.pkl, scaler_mca.pkl, scaler_estrato.pkl
│   └── clasificacion/                 # kmeans_final.pkl, metadata_modelo.pkl
├── reporte.html                       # Reporte automático de sweetviz (generado por 02)
├── LICENSE                            # CC BY-NC-SA 4.0 (código/análisis) + nota de licencia del dataset
├── .gitignore                         # Excluye data/, reporte.html, modelos/**/*.pkl, GUIA_UV.md, entorno virtual, etc.
├── Guia_Ejecucion.md                  # Guía de ejecución específica de este proyecto
└── pyproject.toml / uv.lock           # Dependencias del proyecto
```

> `data/`, `reporte.html` y `modelos/**/*.pkl` no se versionan (ver `.gitignore`) por contener el dataset crudo o contenido derivado de él. Se regeneran corriendo los notebooks en orden desde `01_Limpieza.ipynb`.

## Pipeline

### 1. `01_Limpieza.ipynb`
- Normaliza nombres de columnas y valores de texto (tildes, mayúsculas, espacios).
- Corrige inconsistencias de captura: unifica 210→162 nombres de barrio, corrige 22 registros con fecha de notificación anterior a la del hecho.
- Imputa antecedentes psiquiátricos faltantes (`trastorno_depresivo`, `trastorno_de_personalidad`, `trastorno_bipolar`, `esquizofrenia`) como `NO INFORMADO`.
- Deriva `edad_rango` (7 categorías) y `escolaridad_grupo` (4 niveles), y dicotomiza ~30 variables SI/NO a 0/1.
- Exporta `data/data_clean.csv`.

### 2. `02_Analisis_desciptivo.ipynb`
- Estadística descriptiva diferenciada por tipo de variable (numéricas, categóricas, binarias).
- Matrices de correlación (numéricas y entre factores de riesgo binarios).
- Construye `metodo_principal` a partir de las 8 columnas de método (ahorcamiento, arma cortopunzante, intoxicación, etc.).
- Análisis temporal (casos casi se duplican 2020→2023) y comparación por sexo/edad.


### 3. `03_Preparacion_Clustering.ipynb`
Separa las columnas en tres grupos con roles distintos, clave para la validez de todo lo que sigue:
- **`columnas_modelo`** (35 variables): perfil demográfico y factores de riesgo — entran al clustering.
- **`columnas_validacion`**: `numero_de_intentos` — se reserva para validar los clusters, nunca para formarlos.
- **`columnas_metadata`**: id, fechas, barrio — se conservan para trazabilidad, no participan del modelo.

### 4. `04_Clustering.ipynb` — Modelo final
1. **Depuración de variables** (20 finales): elimina factores binarios raros (<5% prevalencia), y excluye variables con **fuga de información** — `remitido_a_*` (ocurren después del episodio) y las que correlacionan con la variable de validación (`intentos_previos`, `metodo_principal`, `ideacion_suicida_persistente`, `plan_organizado_de_suicidio`). También excluye `sexo`, para que el modelo agrupe por perfil de riesgo y no por género.
2. **Reducción de dimensionalidad**: MCA con corrección de Benzécri (más apropiado que PCA para un espacio dominado por variables categóricas/binarias). Se retienen 4 componentes (80% de varianza corregida) + `estrato_socioeconomico` como eje ordinal estandarizado → espacio final de 5 dimensiones.
3. **Selección de k**: se calculan silueta, Calinski-Harabasz, Davies-Bouldin y el tamaño del cluster más pequeño para k = 2..10. La silueta es prácticamente plana (0.21–0.25, estructura *débil* según Kaufman & Rousseeuw) y su máximo absoluto (k=10) deja un cluster de solo 2.8% de los casos. Exigiendo un tamaño mínimo interpretable (cluster más pequeño ≥5% de los datos), el k final es **k = 9** (silueta = 0.247, mejor Davies-Bouldin que k=10).
4. **Modelo final**: K-Means (k=9, n_init=10) sobre el espacio de 5D, visualizado en 2D vía PCA sobre ese mismo espacio. 
5. **Validación de estabilidad**: 20 reajustes sobre el 90% de los datos → **ARI = 0.856 ± 0.078** (muy estable).
6. **Perfilado**: cada cluster se describe por demografía dominante y factores de riesgo que más se desvían (>15 p.p.) del promedio global; se reporta `numero_de_intentos` y `remitido_a_psiquiatria` (fuera del modelo) como validación externa.
7. **Persistencia**: `MCA`, los dos `StandardScaler` y el `KMeans` final se guardan con `joblib` en `modelos/scaler/` y `modelos/clasificacion/`, junto con la metadata de preprocesamiento — listos para clasificar un caso nuevo sin reajustar el modelo.
8. **Conclusiones**: cierre del notebook con una lectura de los 9 perfiles resultantes (ver `04_Clustering.ipynb`, sección 13, para el detalle completo).

## Limitaciones

- **Estructura de cluster moderada/débil por naturaleza de los datos**: silhouette = 0.247 indica separación débil entre grupos — inherente a datos de salud dominados por variables binarias de baja prevalencia. El modelo es una herramienta de apoyo a la priorización, no una partición exacta ni una etiqueta clínica definitiva por caso.
- No existe todavía un notebook/script de inferencia que cargue los artefactos de `modelos/` para clasificar un caso nuevo end-to-end (los artefactos ya están listos para eso, falta el consumidor).

## Cómo ejecutar

Ver [Guia_Ejecucion.md](Guia_Ejecucion.md) para la guía paso a paso (entorno, descarga del dataset, kernel de Jupyter y orden de los notebooks). Resumen rápido:

```powershell
uv sync
uv run python -m ipykernel install --user --name modelo-salud --display-name "Modelo_Salud (proyecto)"
```

Descargar el dataset en `data/data.csv` (no se versiona, ver [Licencia y uso de datos](#licencia-y-uso-de-datos)) y luego correr los notebooks en orden: `01_Limpieza.ipynb` → `02_Analisis_desciptivo.ipynb` → `03_Preparacion_Clustering.ipynb` → `04_Clustering.ipynb`.
