VENTAS_NL.xlsx

## Taller de Procesamiento de Datos

Machine Learning — Universidad Libre

## Objetivo

Aplicar un flujo completo de procesamiento de datos —limpieza, estandarización e ingeniería de características— sobre la base de datos VENTAS_NL, y a partir de ese análisis, proponer y justificar un modelo de predicción.

## Descripción de la base de datos

La base VENTAS_NL.xlsx contiene registros de ventas con la siguiente estructura:

| Columna | Tipo | Descripción | Observación |
| --- | --- | --- | --- |
| id | Numérico | Identificador único del cliente/transacción | Verificar unicidad |
| EDAD | Numérico | Edad del cliente (años) | Sin datos faltantes |
| GENERO | Categórico | Género del cliente | Validar consistencia de etiquetas |
| SIZE | Numérico/Categórico Talla codificada de la | prenda | Sin datos faltantes |
| YEARINCOME | Numérico | Ingreso anual del cliente | Contiene datos faltantes |
| ventas | Numérico | Valor de la venta | Contiene datos faltantes |
| costo venta | Numérico | Costo asociado a la venta | Contiene datos faltantes |
| DESCUENTOS | Numérico | Proporción de descuento aplicado | Contiene datos faltantes |

## Actividades a desarrollar

## 1. Limpieza de datos

- Determine y elimine los registros duplicados.

- Determine y valide los formatos de cada columna (tipos de dato, rangos válidos, consistencia de categorías); corrija lo que sea necesario.

- Determine si hay datos faltantes en cada columna. Para cada una, calcule el coeficiente de asimetría (skewness) y, con ese criterio, decida el método de imputación más adecuado: media, mediana, moda o regresión. Justifique la elección de cada método con el valor de asimetría obtenido (y, si aplica, con la correlación entre variables).

## 2. Estandarización

- Estandarice YEARINCOME y ventas usando un escalamiento normal (StandardScaler).

- Normalice costo_venta usando un escalamiento uniforme entre 0 y 1 (MinMaxScaler).

## 3. Ingeniería de características


- Plantee al menos dos variables nuevas derivadas de las existentes (por ejemplo, indicadores de margen, ventas netas después de descuento, ingreso normalizado, segmentaciones por edad o por nivel de ingreso, etc.).

- Justifique cada variable propuesta: qué representa y para qué serviría en un análisis o modelo posterior.

## 4. Análisis y propuesta de modelo de predicción

- Con base en la limpieza realizada, analice qué tan relacionadas están las variables entre sí (matriz de correlación).

- Proponga una variable objetivo a predecir y las variables predictoras que usaría, evitando relaciones que generen fuga de datos (data leakage).

- Sugiera un algoritmo apropiado (regresión, árboles, ensamblados, clasificación, etc.) y justifique por qué es adecuado para el problema planteado.

- Realice una prueba exploratoria simple que respalde su propuesta (por ejemplo, un modelo preliminar con su métrica de desempeño).


## Rúbrica de evaluación

| Criterio | Puntos | Qué se evalúa |
| --- | --- | --- |
| Limpieza de datos | 30 | Duplicados, validación de formatos, imputación justificada con asimetría |
| Estandarización | 15 | Aplicación correcta de StandardScaler / MinMaxScaler |
| Ingeniería de características | 20 | Pertinencia y justificación de negocio de las variables nuevas |
| Propuesta de modelo | 25 | Variable objetivo, predictoras, algoritmo y evidencia exploratoria |
| Documentación y claridad | 10 | Comentarios, organización y presentación del trabajo |
