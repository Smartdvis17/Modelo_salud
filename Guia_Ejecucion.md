# Guía de ejecución — Modelo_Salud

Guía específica para reproducir este proyecto (clustering de perfiles de riesgo de intento de suicidio). No es una guía genérica de `uv`; asume que ya tienes `uv` instalado.

## 1. Clonar y sincronizar el entorno

```powershell
git clone <url-de-este-repo>
cd Modelo_Salud
uv sync
```

`uv sync` crea `.venv/` e instala exactamente las dependencias fijadas en `uv.lock` (pandas, scikit-learn, prince, sweetviz, statsmodels, seaborn, matplotlib, joblib).

## 2. Descargar el dataset (no incluido en el repo)

`data/` está excluido del repositorio vía `.gitignore` — ver [Licencia y uso de datos](README.md#licencia-y-uso-de-datos) en el README. Para reproducir el pipeline:

1. Descarga el CSV desde [datos.gov.co — Información Intentos de Suicidio Municipio de Tunja, Boyacá](https://www.datos.gov.co/Salud-y-Protecci-n-Social/Informaci-n-Intentos-de-Suicidio-Municipio-de-Tunj/nk8x-s9hw/about_data) (botón "Exportar" → CSV).
2. Colócalo en `data/data.csv` (crea la carpeta `data/` si no existe). Los demás archivos de `data/` (`data_clean.csv`, `data_cluster.csv`, `data_validacion.csv`) los genera el propio pipeline al ejecutar los notebooks.

## 3. Registrar el kernel de Jupyter de este proyecto

```powershell
uv run python -m ipykernel install --user --name modelo-salud --display-name "Modelo_Salud (proyecto)"
```

Al abrir cualquiera de los 4 notebooks en VS Code / Jupyter, selecciona explícitamente el kernel **"Modelo_Salud (proyecto)"**. No uses un kernel genérico llamado "python3": si tienes más de un proyecto Python en la máquina, ese nombre puede quedar ambiguo o apuntar al entorno de otro proyecto, y las celdas fallan con `ModuleNotFoundError` aunque `uv sync` se haya ejecutado correctamente aquí.

## 4. Ejecutar los notebooks en orden

Cada notebook depende del CSV que produce el anterior — deben correrse en este orden, de principio a fin ("Run All"):

| # | Notebook | Produce |
|---|---|---|
| 1 | `01_Limpieza.ipynb` | `data/data_clean.csv` |
| 2 | `02_Analisis_desciptivo.ipynb` | `reporte.html` (EDA; no genera CSV) |
| 3 | `03_Preparacion_Clustering.ipynb` | `data/data_cluster.csv`, `data/data_validacion.csv` |
| 4 | `04_Clustering.ipynb` | Perfiles de riesgo + `modelos/scaler/*.pkl`, `modelos/clasificacion/*.pkl` |

## 5. Verificar que corrió bien

Al final de `04_Clustering.ipynb` deberías ver:
- La celda de selección de k imprimiendo `k elegido: 6 (silueta = 0.383)`.
- La celda de estabilidad imprimiendo un ARI promedio cercano a `0.977`.
- La celda de persistencia listando 4 archivos `.pkl`: `modelos/scaler/mca.pkl`, `modelos/scaler/scaler_mca.pkl`, `modelos/clasificacion/kmeans_final.pkl`, `modelos/clasificacion/metadata_modelo.pkl`.

## Problemas comunes

- **`ModuleNotFoundError` al ejecutar una celda**: el notebook está usando el kernel equivocado (ver paso 3), no que falte instalar algo — `uv sync` ya instaló todo lo necesario en `.venv/`.
- **`FileNotFoundError: data/data.csv`**: falta el paso 2 (el dataset no se versiona en git).
- **Los notebooks 02–04 fallan**: deben ejecutarse en orden desde `01_Limpieza.ipynb`; cada uno espera el CSV de salida del anterior en `data/`.
