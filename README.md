# Modelo Salud — Clustering de perfiles de riesgo

Proyecto de aprendizaje no supervisado que segmenta los casos de intento de suicidio del municipio de Tunja, Boyacá, en perfiles de riesgo mediante **MCA (Análisis de Correspondencias Múltiples) + K-Means**.

## Fuente de datos

- **Dataset**: Información Intentos de Suicidio, Municipio de Tunja, Boyacá.
- **Origen**: [datos.gov.co](https://www.datos.gov.co/Salud-y-Protecci-n-Social/Informaci-n-Intentos-de-Suicidio-Municipio-de-Tunj/nk8x-s9hw/about_data) (dato público).
- **Tamaño**: 818 registros, 48 columnas originales (2020 – marzo 2024).

> **Nota de manejo responsable de datos**: es información sensible de salud mental. El dataset es público y no incluye identificadores directos (nombre, documento), pero sí variables cuasi-identificadoras (barrio de residencia, fecha, edad). Este repositorio se usa con fines de portafolio; cualquier uso derivado para decisiones reales de política pública debe pasar por un comité de ética y anonimización adicional.

## Licencia y uso de datos

**Dataset**: licencia [Creative Commons Atribución-CompartirIgual 4.0 Internacional (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/legalcode), atribución **Alcaldía de Tunja, Boyacá** (verificado en el metadato oficial del dataset en datos.gov.co). Este repositorio no redistribuye el dataset (ver `.gitignore`); solo lo referencia y lo transforma para producir el análisis publicado aquí.

**Código, notebooks y análisis propios**: © Ana Salcedo Martínez, licencia [Creative Commons Atribución-NoComercial-CompartirIgual 4.0 (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode) — ver texto completo en [LICENSE](LICENSE). 

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
│   ├── scaler/                        # mca.pkl, scaler_mca.pkl
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
1. **Depuración de variables** (17 finales), se elimina factores binarios raros (<5% prevalencia); excluye `remitido_a_*` por **fuga de información temporal** (ocurren después del episodio); excluye `intentos_previos`, `metodo_principal`, `ideacion_suicida_persistente` y `plan_organizado_de_suicidio` por ser **indicadores del propio episodio** (el mismo constructo que mide la variable de validación `numero_de_intentos`) y no factores de riesgo antecedentes — de forma consistente con esto, también son las variables más correlacionadas con `numero_de_intentos`, lo que confirma la decisión pero no la origina; excluye `sexo` para que el modelo agrupe por perfil de riesgo y no por género; excluye `area_de_residencia` por ser casi constante (98.3% en una sola categoría, no puede separar perfiles por definición); y excluye `estrato_socioeconomico` y `seguridad_social` por ser variables estructurales de acceso socioeconómico, no factores de riesgo psicosocial. 
2. **Reducción de dimensionalidad**: MCA con corrección de Benzécri (más apropiado que PCA para un espacio dominado por variables categóricas/binarias) sobre las 17 variables restantes, todas categóricas/binarias. Se retienen 3 componentes (80% de varianza corregida) → espacio final de 3 dimensiones, puramente MCA (sin ninguna variable numérica/ordinal añadida aparte).
3. **Selección de k**: se calculan silueta, y se escoge el tamaño del cluster más pequeño para k = 2..10. La silueta se mueve entre 0.330 y 0.383, con **k=6 como máximo absoluto** — y ese mismo k ya cumple la regla de tamaño mínimo de cluster (7.8% ≥ 5%), así que el óptimo estadístico y el accionable coinciden.
4. **Modelo final**: K-Means (k=6, n_init=10) sobre el espacio de 3D, visualizado en 2D vía PCA sobre ese mismo espacio.
5. **Validación de estabilidad**: 20 reajustes sobre el 90% de los datos → **ARI = 0.977 ± 0.025** (muy estable). Esto valida que la partición es *reproducible* frente a submuestreo, no que existan 6 grupos naturalmente discretos en la población — esa pregunta la responde la silueta (punto 3), que se mantiene en la banda de "estructura débil" (ver Limitaciones).
6. **Perfilado**: cada cluster se describe por demografía dominante y factores de riesgo que más se desvían (>15 p.p.) del promedio global; se reportan `numero_de_intentos`, `remitido_a_psiquiatria`, `estrato_socioeconomico` y `seguridad_social` (las cuatro fuera del modelo) como validación/caracterización externa.
7. **Persistencia**: `MCA`, el `StandardScaler` de los componentes de MCA y el `KMeans` final se guardan con `joblib` en `modelos/scaler/` y `modelos/clasificacion/`, junto con la metadata de preprocesamiento — listos para clasificar un caso nuevo sin reajustar el modelo.
8. **Conclusiones**: cierre del notebook con una lectura de los 6 perfiles resultantes (ver `04_Clustering.ipynb`, sección 13, para el detalle completo).

## Limitaciones

- **Estructura de cluster moderada por naturaleza de los datos**: silhouette = 0.383 mejoró sustancialmente frente a versiones anteriores (0.247 → 0.260 → 0.383) al depurar variables que diluían la señal de riesgo, pero sigue por debajo del umbral de "estructura fuerte" (>0.5) y dentro de la banda de "estructura débil",  Los 6 perfiles deben leerse como una segmentación reproducible y útil para priorización, no como categorías naturales discretas ni como una clasificación diagnóstica de individuos.


## Cómo ejecutar

Ver [Guia_Ejecucion.md](Guia_Ejecucion.md) para la guía paso a paso (entorno, descarga del dataset, kernel de Jupyter y orden de los notebooks). Resumen rápido:

```powershell
uv sync
uv run python -m ipykernel install --user --name modelo-salud --display-name "Modelo_Salud (proyecto)"
```

Descargar el dataset en `data/data.csv` (no se versiona, ver [Licencia y uso de datos](#licencia-y-uso-de-datos)) y luego correr los notebooks en orden: `01_Limpieza.ipynb` → `02_Analisis_desciptivo.ipynb` → `03_Preparacion_Clustering.ipynb` → `04_Clustering.ipynb`.
