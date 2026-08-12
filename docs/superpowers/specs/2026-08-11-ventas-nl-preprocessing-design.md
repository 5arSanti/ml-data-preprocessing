# Design: Preprocesamiento VENTAS_NL (`main.ipynb`)

Fecha: 2026-08-11  
Enunciado: `Taller_Procesamiento_VENTAS_NL_1.md`  
Guía de referencia: `scripts/Taller_Preprocesamiento_(Minería_de_Datos).ipynb`  
Datos: `data_source.xlsx` (equivalente a `VENTAS_NL.xlsx` del enunciado)

## 1. Objetivo

Entregar un Jupyter narrativo completo (`main.ipynb` en la raíz del proyecto) que aplique el flujo del taller:

1. Limpieza de datos  
2. Estandarización  
3. Ingeniería de características  
4. Análisis, propuesta de modelo y prueba exploratoria  

Cada celda de código tiene una responsabilidad única y una celda Markdown previa que explica, en español, el propósito del paso y cómo interpretar los resultados.

## 2. Decisiones confirmadas

| Tema | Decisión |
| --- | --- |
| Alcance | Actividades 1–4 completas (rúbrica completa) |
| Datos faltantes | Solo imputación justificada por asimetría; no eliminar filas por nulos |
| Duplicados por `id` | Eliminar (`keep='first'`) |
| Duplicados por estructura | Conservar y marcar con flag `dup_estructura` |
| Duplicados significativos | Conservar y marcar con flag `dup_significativo` |
| Incoherencias | Corregir cuando la regla sea clara; flaggear lo que no se pueda corregir de forma segura |
| Target del modelo | `ventas` (regresión) |
| Enfoque de entrega | Notebook narrativo por secciones del taller (no módulo `.py` separado, no Pipeline opaco) |
| Archivo entregable | `main.ipynb` en la raíz del repositorio |

## 3. Arquitectura del notebook

### 3.1 Archivo y fuentes

- **Entregable:** `main.ipynb` (raíz del repositorio)
- **Entrada:** `data_source.xlsx`
- **Columnas origen:** `id`, `EDAD`, `GENERO`, `SIZE`, `YEARINCOME`, `ventas`, `costo venta`, `DESCUENTOS`
- **Rename documentado:** `costo venta` → `costo_venta`

### 3.2 Secciones

1. Introducción y carga  
2. Exploración inicial (shape, dtypes, nulos, estadísticas)  
3. Limpieza  
4. Estandarización  
5. Ingeniería de características  
6. Análisis y modelo  
7. Conclusiones  

### 3.3 Convención de celdas

- Markdown: qué se hace, por qué, criterio usado, cómo leer la salida.  
- Código: una acción principal por celda (conteo, transformación, gráfico o métrica).  
- Tras transformaciones relevantes: imprimir conteos antes/después o resúmenes de validación.

### 3.4 Dependencias previstas

`pandas`, `numpy`, `matplotlib`/`seaborn`, `scikit-learn`, `openpyxl` (lectura Excel).

## 4. Limpieza de datos

### 4.1 Duplicados (tres niveles)

1. **Global por `id`:** contar duplicados → eliminar con `drop_duplicates(subset=['id'], keep='first')` → reportar filas removidas.  
2. **Estructura:** comparar todas las columnas excepto `id`. **No eliminar.** Crear `dup_estructura` (bool).  
3. **Variables significativas:** subset `['EDAD', 'GENERO', 'SIZE', 'YEARINCOME']`. **No eliminar.** Crear `dup_significativo` (bool).

### 4.2 Formatos, rangos y categorías

- Forzar tipos numéricos en columnas numéricas; reportar valores no convertibles.  
- `GENERO`: validar/unificar etiquetas; valores observados esperados: `M`, `H`.  
- `SIZE`: entero en {1, 2, 3, 4, 5}.  
- `EDAD`: rango plausible (p. ej. 18–100); documentar extremos reales del dataset.  
- `DESCUENTOS`: proporción en [0, 1].

### 4.3 Coherencia matemática y de negocio

| Regla | Acción |
| --- | --- |
| `YEARINCOME` < 0 | Clip a 0 + `flag_neg_YEARINCOME` |
| `ventas` < 0 | Clip a 0 + `flag_neg_ventas` |
| `costo_venta` < 0 | Clip a 0 + `flag_neg_costo_venta` |
| `DESCUENTOS` < 0 | Clip a 0 + `flag_neg_DESCUENTOS` |
| `DESCUENTOS` > 1 | Clip a 1 + `flag_DESCUENTOS_gt_1` |
| `costo_venta > ventas` (ambos no nulos) | **Solo flag** `flag_costo_gt_ventas` (sin corrección automática insegura) |

Tras imputación, revalidar no-negatividad y rangos.

### 4.4 Imputación por asimetría

Columnas con nulos esperados: `YEARINCOME`, `ventas`, `costo_venta`, `DESCUENTOS`.

Por cada columna numérica con nulos:

1. Calcular coeficiente de asimetría (skewness) sobre valores no nulos.  
2. Criterio de decisión:  
   - `|skew| < 0.5` → media  
   - `|skew| ≥ 0.5` → mediana  
3. Regresión: solo si se justifica con correlación fuerte y predictores disponibles; documentar en Markdown.  
4. No eliminar filas por presencia de nulos.  
5. Justificar en Markdown el valor de asimetría y el método elegido.

## 5. Estandarización

Aplicar sobre el dataset ya limpio e imputado:

| Variable | Método | Columnas nuevas |
| --- | --- | --- |
| `YEARINCOME` | `StandardScaler` | `YEARINCOME_std` |
| `ventas` | `StandardScaler` | `ventas_std` |
| `costo_venta` | `MinMaxScaler` | `costo_venta_minmax` |

- Conservar columnas originales.  
- En esta sección pedagógica, mostrar `fit_transform` y verificar media≈0 / std≈1 o min≈0 / max≈1.  
- En la sección de modelo, cualquier escalado de **predictoras** se ajusta solo en train para evitar fuga.

## 6. Ingeniería de características

| Variable | Definición | Uso analítico | Predictor de `ventas` |
| --- | --- | --- | --- |
| `ventas_netas` | `ventas * (1 - DESCUENTOS)` | Valor efectivo tras descuento | No (leakage) |
| `margen_bruto` | `ventas - costo_venta` | Rentabilidad absoluta | No (leakage) |
| `grupo_edad` | bins de `EDAD` (p. ej. 21–30, 31–40, 41–55) | Segmentación demográfica | Sí |
| `segmento_ingreso` | cuantiles de `YEARINCOME` (bajo/medio/alto) | Poder adquisitivo relativo | Sí |

Cada variable se justifica en Markdown (qué representa y para qué sirve en análisis o modelado).

## 7. Análisis y propuesta de modelo

### 7.1 Correlación

- Matriz de correlación de variables numéricas (incluye derivadas).  
- Heatmap + interpretación de relaciones y riesgo de colinealidad.

### 7.2 Problema de predicción

- **Target:** `ventas` (escala original).  
- **Predictoras candidatas:** `EDAD`, `GENERO`, `SIZE`, `YEARINCOME`, `DESCUENTOS`, `grupo_edad`, `segmento_ingreso`.  
- **Excluidas por leakage:** `ventas_netas`, `margen_bruto`, `ventas_std`, y cualquier transformación que use `ventas`.  
- **`costo_venta`:** excluida del baseline por cuasi-leakage (costo y venta suelen co-determinarse). Documentar la decisión.

### 7.3 Algoritmo y evidencia

- Baseline: `RandomForestRegressor` (no linealidad + mezcla de tipos tras encoding).  
- Referencia opcional: regresión lineal.  
- Encoding de categóricas (`GENERO`, `grupo_edad`, `segmento_ingreso`).  
- Split train/test 80/20 con `random_state` fijo.  
- Métricas en test: RMSE, MAE, R².  
- Importancia de variables (RF) y comentario breve de residuales.  
- Conclusión: adecuación del enfoque y posibles mejoras.

## 8. Manejo de errores y trazabilidad

- Reportar conteos en cada etapa (filas, nulos, duplicados, flags).  
- No silenciar fallos de conversión de tipo: mostrarlos y decidir corrección documentada.  
- Flags booleanos permanecen en el DataFrame para auditoría; no sustituyen la justificación en Markdown.

## 9. Criterios de éxito

- Cumple las cuatro actividades del enunciado.  
- Duplicados: elimina solo por `id`; flags para estructura y significativos.  
- Imputación justificada con skewness por columna; sin drop por nulos.  
- Scalers aplicados según enunciado, con columnas nuevas y verificación numérica.  
- ≥2 features nuevas con justificación; leakage documentado en el modelo.  
- Modelo exploratorio de `ventas` con métricas e interpretación.  
- Narrativa MD+código coherente, en español, una responsabilidad por celda.

## 10. Fuera de alcance

- API, UI o pipeline de producción.  
- Extracción de lógica a módulos `.py` (salvo que se decida después).  
- Optimización exhaustiva de hiperparámetros.  
- Sustitución del dataset o renombrado obligatorio a `VENTAS_NL.xlsx` (se documenta el alias `data_source.xlsx`).
