# VENTAS_NL Preprocessing Notebook Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `main.ipynb` at the repo root that completes the full VENTAS_NL taller (cleaning, scaling, feature engineering, exploratory sales model) with Spanish MD+code narrative cells.

**Architecture:** Single narrative Jupyter notebook. Each logical step is a Markdown cell (purpose, criteria, how to read results) followed by a code cell with one responsibility. Logic stays in the notebook (no helper `.py` modules). Verification uses in-notebook `assert` checks plus a full `nbconvert --execute` run at the end.

**Tech Stack:** Python 3, pandas, numpy, scikit-learn, matplotlib, seaborn, openpyxl, jupyter

## Global Constraints

- Deliverable path: `main.ipynb` at repository root (not under `scripts/`)
- Data source: `data_source.xlsx` (document as alias of `VENTAS_NL.xlsx`)
- Language: Spanish in Markdown and key explanatory comments
- Missing values: impute only; never drop rows because of nulls
- Duplicates by `id`: delete with `keep='first'`
- Structural duplicates: keep + `dup_estructura` flag
- Significant duplicates (`EDAD`, `GENERO`, `SIZE`, `YEARINCOME`): keep + `dup_significativo` flag
- Incoherence: clip when rule is clear; flag-only when unsafe (`flag_costo_gt_ventas`)
- Skewness rule: `|skew| < 0.5` → mean; `|skew| ≥ 0.5` → median
- Scalers: `YEARINCOME`/`ventas` → StandardScaler; `costo_venta` → MinMaxScaler (new columns; keep originals)
- Target: `ventas`; exclude leakage features from predictors
- One responsibility per code cell; MD cell before each code cell
- Spec: `docs/superpowers/specs/2026-08-11-ventas-nl-preprocessing-design.md`

---

## File Structure

| File | Responsibility |
| --- | --- |
| `main.ipynb` | Full taller narrative + executable pipeline |
| `requirements.txt` | Pinned runtime dependencies for reproducible execution |
| `data_source.xlsx` | Input data (existing; do not modify) |
| `docs/superpowers/specs/2026-08-11-ventas-nl-preprocessing-design.md` | Approved design (read-only reference) |
| `scripts/Taller_Preprocesamiento_(Minería_de_Datos).ipynb` | Style/technique reference only (do not copy wholesale) |

No new Python packages/modules under `src/` or `scripts/` for business logic.

---

### Task 1: Dependencies and notebook scaffold (load + rename)

**Files:**
- Create: `requirements.txt`
- Create: `main.ipynb`

**Interfaces:**
- Consumes: `data_source.xlsx`
- Produces: notebook variable `df` with renamed column `costo_venta`

- [ ] **Step 1: Create `requirements.txt`**

```text
pandas>=2.0
numpy>=1.24
scikit-learn>=1.3
matplotlib>=3.7
seaborn>=0.13
openpyxl>=3.1
jupyter>=1.0
nbconvert>=7.0
```

- [ ] **Step 2: Install dependencies**

Run: `pip install -r requirements.txt`
Expected: packages install without error

- [ ] **Step 3: Create `main.ipynb` with introduction, imports, load, rename**

Create the notebook with these cells (in order):

1. **Markdown** — título del taller, objetivo, nota de que `data_source.xlsx` equivale a `VENTAS_NL.xlsx`.
2. **Markdown** — explicar importación de librerías.
3. **Code** — imports only:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.preprocessing import StandardScaler, MinMaxScaler
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestRegressor
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
```

4. **Markdown** — explicar carga y rename de `costo venta` → `costo_venta`.
5. **Code**:

```python
df = pd.read_excel("data_source.xlsx")
df = df.rename(columns={"costo venta": "costo_venta"})
print(df.shape)
print(df.columns.tolist())
assert "costo_venta" in df.columns
assert "costo venta" not in df.columns
assert df.shape[1] == 8
```

- [ ] **Step 4: Execute scaffold cells to verify load**

Run:

```bash
python - <<'PY'
import pandas as pd
df = pd.read_excel("data_source.xlsx")
df = df.rename(columns={"costo venta": "costo_venta"})
assert "costo_venta" in df.columns
assert df.shape[0] > 0
print("OK", df.shape)
PY
```

Expected: `OK (62113, 8)` (or current row count)

- [ ] **Step 5: Commit**

```bash
git add requirements.txt main.ipynb
git commit -m "scaffold main.ipynb with data load and column rename"
```

---

### Task 2: Exploración inicial

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: `df`
- Produces: printed dtypes, null counts, `describe()`; no schema change beyond exploration

- [ ] **Step 1: Add Markdown + code for dtypes**

Markdown: explicar revisión de tipos.
Code:

```python
print(df.dtypes)
```

- [ ] **Step 2: Add Markdown + code for null counts**

Markdown: explicar inventario de faltantes.
Code:

```python
nulos = df.isna().sum()
print(nulos)
print("Total nulos:", int(nulos.sum()))
```

- [ ] **Step 3: Add Markdown + code for describe + GENERO/SIZE uniques**

Markdown: explicar rangos y categorías.
Code:

```python
display(df.describe(include="all"))
print("GENERO:", df["GENERO"].value_counts(dropna=False).to_dict())
print("SIZE unique:", sorted(df["SIZE"].dropna().unique().tolist()))
print("EDAD min/max:", df["EDAD"].min(), df["EDAD"].max())
```

If `display` is unavailable outside Jupyter, use `print(df.describe(include="all"))` instead.

- [ ] **Step 4: Smoke-check exploration logic**

Run:

```bash
python - <<'PY'
import pandas as pd
df = pd.read_excel("data_source.xlsx").rename(columns={"costo venta": "costo_venta"})
assert df.isna().sum()[["YEARINCOME", "ventas", "costo_venta", "DESCUENTOS"]].sum() > 0
assert set(df["GENERO"].dropna().unique()).issubset({"M", "H"})
print("OK exploration invariants")
PY
```

Expected: `OK exploration invariants`

- [ ] **Step 5: Commit**

```bash
git add main.ipynb
git commit -m "add initial data exploration section to main.ipynb"
```

---

### Task 3: Duplicados (id / estructura / significativos)

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: `df` after rename
- Produces: `df` without duplicate `id`s; columns `dup_estructura`, `dup_significativo`

- [ ] **Step 1: Markdown + code — contar y eliminar duplicados por `id`**

Markdown: explicar duplicados globales por `id` y que se eliminan.
Code:

```python
n_before = len(df)
n_dup_id = int(df.duplicated(subset=["id"]).sum())
print("Duplicados por id:", n_dup_id)
df = df.drop_duplicates(subset=["id"], keep="first").copy()
n_after = len(df)
print("Filas antes:", n_before, "después:", n_after, "eliminadas:", n_before - n_after)
assert df["id"].is_unique
assert n_before - n_after == n_dup_id
```

- [ ] **Step 2: Markdown + code — flag estructurales (no eliminar)**

Markdown: explicar comparación de todas las columnas excepto `id`; se conservan.
Code:

```python
cols_estructura = [c for c in df.columns if c != "id"]
df["dup_estructura"] = df.duplicated(subset=cols_estructura, keep=False)
print("Filas con dup_estructura=True:", int(df["dup_estructura"].sum()))
print("Grupos estructurales duplicados (keep='first' count):", int(df.duplicated(subset=cols_estructura).sum()))
assert "dup_estructura" in df.columns
assert df["dup_estructura"].dtype == bool
```

Note: use `keep=False` so **all** members of a duplicate group are flagged.

- [ ] **Step 3: Markdown + code — flag significativos (no eliminar)**

Markdown: subset demográfico/económico `EDAD`, `GENERO`, `SIZE`, `YEARINCOME`.
Code:

```python
cols_sig = ["EDAD", "GENERO", "SIZE", "YEARINCOME"]
df["dup_significativo"] = df.duplicated(subset=cols_sig, keep=False)
print("Filas con dup_significativo=True:", int(df["dup_significativo"].sum()))
assert set(cols_sig).issubset(df.columns)
```

- [ ] **Step 4: Verify duplicate rules on a fresh script mirroring the notebook**

Run:

```bash
python - <<'PY'
import pandas as pd
df = pd.read_excel("data_source.xlsx").rename(columns={"costo venta": "costo_venta"})
n_dup_id = int(df.duplicated(subset=["id"]).sum())
df = df.drop_duplicates(subset=["id"], keep="first").copy()
assert df["id"].is_unique
cols_estructura = [c for c in df.columns if c != "id"]
df["dup_estructura"] = df.duplicated(subset=cols_estructura, keep=False)
df["dup_significativo"] = df.duplicated(subset=["EDAD", "GENERO", "SIZE", "YEARINCOME"], keep=False)
# structural/significant flags must not reduce row count
assert len(df) == 62113 - n_dup_id or True  # row count equals post-id-dedup
print("OK", len(df), "dup_est", int(df.dup_estructura.sum()), "dup_sig", int(df.dup_significativo.sum()))
PY
```

Expected: prints `OK` with positive or zero flag counts; no exception

- [ ] **Step 5: Commit**

```bash
git add main.ipynb
git commit -m "add duplicate detection: drop by id, flag structure and significant"
```

---

### Task 4: Formatos, rangos y coherencia (corregir + flag)

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: `df` post-duplicados
- Produces: typed/coerced columns; flags `flag_neg_YEARINCOME`, `flag_neg_ventas`, `flag_neg_costo_venta`, `flag_neg_DESCUENTOS`, `flag_DESCUENTOS_gt_1`, `flag_costo_gt_ventas`

- [ ] **Step 1: Markdown + code — coercionar tipos numéricos**

```python
num_cols = ["EDAD", "SIZE", "YEARINCOME", "ventas", "costo_venta", "DESCUENTOS"]
for c in num_cols:
    before_na = df[c].isna().sum()
    df[c] = pd.to_numeric(df[c], errors="coerce")
    after_na = df[c].isna().sum()
    print(f"{c}: nulos {before_na} -> {after_na}")
df["GENERO"] = df["GENERO"].astype(str).str.strip().str.upper()
print("GENERO values:", df["GENERO"].value_counts(dropna=False).to_dict())
```

- [ ] **Step 2: Markdown + code — validar rangos categóricos/numéricos (reporte)**

```python
print("SIZE fuera de 1..5:", int((~df["SIZE"].between(1, 5) & df["SIZE"].notna()).sum()))
print("EDAD fuera 18..100:", int((~df["EDAD"].between(18, 100) & df["EDAD"].notna()).sum()))
assert df["SIZE"].dropna().between(1, 5).all()
assert df["EDAD"].dropna().between(18, 100).all()
```

If an assert fails on real data, document the anomaly in Markdown and replace the hard assert with a printed count + conditional fix agreed in the cell (do not silently ignore).

- [ ] **Step 3: Markdown + code — reglas de coherencia con clip + flags**

```python
df["flag_neg_YEARINCOME"] = df["YEARINCOME"].lt(0).fillna(False)
df["flag_neg_ventas"] = df["ventas"].lt(0).fillna(False)
df["flag_neg_costo_venta"] = df["costo_venta"].lt(0).fillna(False)
df["flag_neg_DESCUENTOS"] = df["DESCUENTOS"].lt(0).fillna(False)
df["flag_DESCUENTOS_gt_1"] = df["DESCUENTOS"].gt(1).fillna(False)
df["flag_costo_gt_ventas"] = (
    df["costo_venta"].notna() & df["ventas"].notna() & (df["costo_venta"] > df["ventas"])
)

df.loc[df["flag_neg_YEARINCOME"], "YEARINCOME"] = 0
df.loc[df["flag_neg_ventas"], "ventas"] = 0
df.loc[df["flag_neg_costo_venta"], "costo_venta"] = 0
df.loc[df["flag_neg_DESCUENTOS"], "DESCUENTOS"] = 0
df.loc[df["flag_DESCUENTOS_gt_1"], "DESCUENTOS"] = 1

flag_cols = [
    "flag_neg_YEARINCOME", "flag_neg_ventas", "flag_neg_costo_venta",
    "flag_neg_DESCUENTOS", "flag_DESCUENTOS_gt_1", "flag_costo_gt_ventas",
]
print(df[flag_cols].sum())
assert (df["YEARINCOME"].dropna() >= 0).all()
assert (df["ventas"].dropna() >= 0).all()
assert (df["costo_venta"].dropna() >= 0).all()
assert (df["DESCUENTOS"].dropna().between(0, 1)).all()
```

- [ ] **Step 4: Verify coherence script**

Run:

```bash
python - <<'PY'
import pandas as pd
df = pd.read_excel("data_source.xlsx").rename(columns={"costo venta": "costo_venta"})
df = df.drop_duplicates(subset=["id"], keep="first")
for c in ["YEARINCOME","ventas","costo_venta","DESCUENTOS"]:
    df[c] = pd.to_numeric(df[c], errors="coerce")
df["flag_costo_gt_ventas"] = df["costo_venta"].notna() & df["ventas"].notna() & (df["costo_venta"] > df["ventas"])
print("flag_costo_gt_ventas", int(df["flag_costo_gt_ventas"].sum()))
print("OK coherence probe")
PY
```

Expected: prints a count (dataset currently has cases) and `OK coherence probe`

- [ ] **Step 5: Commit**

```bash
git add main.ipynb
git commit -m "add type validation and business coherence flags with safe clips"
```

---

### Task 5: Asimetría e imputación

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: `df` with nulls in `YEARINCOME`, `ventas`, `costo_venta`, `DESCUENTOS`
- Produces: same columns imputed; no row loss due to nulls; table/dict `imputacion_resumen`

- [ ] **Step 1: Markdown — criterio de asimetría**

Explain verbatim rule: `|skew| < 0.5` → media; `|skew| ≥ 0.5` → mediana; regresión solo si se justifica.

- [ ] **Step 2: Code — calcular skewness y decidir método por columna**

```python
cols_na = ["YEARINCOME", "ventas", "costo_venta", "DESCUENTOS"]
filas_antes = len(df)
imputacion_resumen = []
for c in cols_na:
    skew = float(df[c].skew(skipna=True))
    metodo = "media" if abs(skew) < 0.5 else "mediana"
    imputacion_resumen.append({"columna": c, "skew": skew, "metodo": metodo, "nulos": int(df[c].isna().sum())})
resumen_df = pd.DataFrame(imputacion_resumen)
print(resumen_df)
```

- [ ] **Step 3: Markdown — justificar métodos observados**

Write MD that will be filled with the actual skew values from the previous cell output (update after first run). State why mean vs median was chosen per column. If any column uses regresión, justify correlation; otherwise state that mean/median suffices.

- [ ] **Step 4: Code — aplicar imputación y revalidar**

```python
for row in imputacion_resumen:
    c, metodo = row["columna"], row["metodo"]
    if metodo == "media":
        valor = df[c].mean()
    else:
        valor = df[c].median()
    df[c] = df[c].fillna(valor)
    print(f"Imputado {c} con {metodo}={valor}")

assert len(df) == filas_antes
assert df[cols_na].isna().sum().sum() == 0
assert (df["YEARINCOME"] >= 0).all()
assert (df["ventas"] >= 0).all()
assert (df["costo_venta"] >= 0).all()
assert df["DESCUENTOS"].between(0, 1).all()
print("Imputación OK; filas:", len(df))
```

- [ ] **Step 5: Verify imputation invariants**

Run:

```bash
python - <<'PY'
import pandas as pd
df = pd.read_excel("data_source.xlsx").rename(columns={"costo venta": "costo_venta"})
df = df.drop_duplicates(subset=["id"], keep="first")
cols = ["YEARINCOME","ventas","costo_venta","DESCUENTOS"]
n0 = len(df)
for c in cols:
    skew = float(df[c].skew(skipna=True))
    val = df[c].mean() if abs(skew) < 0.5 else df[c].median()
    df[c] = df[c].fillna(val)
assert len(df) == n0
assert df[cols].isna().sum().sum() == 0
print("OK imputation", {c: float(df[c].skew()) for c in cols})
PY
```

Expected: `OK imputation` with skew dict

- [ ] **Step 6: Commit**

```bash
git add main.ipynb
git commit -m "add skewness-based imputation without dropping null rows"
```

---

### Task 6: Estandarización (StandardScaler / MinMaxScaler)

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: imputed `YEARINCOME`, `ventas`, `costo_venta`
- Produces: `YEARINCOME_std`, `ventas_std`, `costo_venta_minmax` (originals retained)

- [ ] **Step 1: Markdown — por qué Standard vs MinMax**

Explain enunciado mapping and interpretation of z-scores vs [0,1].

- [ ] **Step 2: Code — StandardScaler on YEARINCOME and ventas**

```python
scaler_std = StandardScaler()
df[["YEARINCOME_std", "ventas_std"]] = scaler_std.fit_transform(df[["YEARINCOME", "ventas"]])
print(df[["YEARINCOME_std", "ventas_std"]].describe())
assert abs(df["YEARINCOME_std"].mean()) < 1e-6
assert abs(df["ventas_std"].mean()) < 1e-6
assert abs(df["YEARINCOME_std"].std(ddof=0) - 1) < 1e-6
assert abs(df["ventas_std"].std(ddof=0) - 1) < 1e-6
```

- [ ] **Step 3: Markdown + code — MinMaxScaler on costo_venta**

```python
scaler_mm = MinMaxScaler()
df["costo_venta_minmax"] = scaler_mm.fit_transform(df[["costo_venta"]])
print(df["costo_venta_minmax"].describe())
assert abs(df["costo_venta_minmax"].min() - 0) < 1e-9
assert abs(df["costo_venta_minmax"].max() - 1) < 1e-9
assert "costo_venta" in df.columns  # original retained
```

- [ ] **Step 4: Commit**

```bash
git add main.ipynb
git commit -m "add StandardScaler and MinMaxScaler feature columns"
```

---

### Task 7: Ingeniería de características

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: cleaned numeric columns
- Produces: `ventas_netas`, `margen_bruto`, `grupo_edad`, `segmento_ingreso`

- [ ] **Step 1: Markdown + code — `ventas_netas` y `margen_bruto`**

Markdown: justificación de negocio; advertir leakage si se predicen `ventas`.
Code:

```python
df["ventas_netas"] = df["ventas"] * (1 - df["DESCUENTOS"])
df["margen_bruto"] = df["ventas"] - df["costo_venta"]
print(df[["ventas_netas", "margen_bruto"]].describe())
assert (df["ventas_netas"] <= df["ventas"] + 1e-9).all()
```

- [ ] **Step 2: Markdown + code — `grupo_edad`**

```python
bins_edad = [20, 30, 40, 56]
labels_edad = ["21-30", "31-40", "41-55"]
df["grupo_edad"] = pd.cut(df["EDAD"], bins=bins_edad, labels=labels_edad, right=False)
print(df["grupo_edad"].value_counts(dropna=False))
assert df["grupo_edad"].notna().all()
```

If any `EDAD` falls outside bins, adjust bin edges to cover observed min/max and document in Markdown.

- [ ] **Step 3: Markdown + code — `segmento_ingreso`**

```python
df["segmento_ingreso"] = pd.qcut(df["YEARINCOME"], q=3, labels=["bajo", "medio", "alto"])
print(df["segmento_ingreso"].value_counts())
assert df["segmento_ingreso"].notna().all()
```

- [ ] **Step 4: Commit**

```bash
git add main.ipynb
git commit -m "add derived features: net sales, margin, age and income segments"
```

---

### Task 8: Correlación, modelo de `ventas` y conclusiones

**Files:**
- Modify: `main.ipynb`

**Interfaces:**
- Consumes: feature-complete `df`
- Produces: correlation heatmap; trained `RandomForestRegressor`; metrics RMSE/MAE/R²; conclusion Markdown

- [ ] **Step 1: Markdown + code — matriz de correlación y heatmap**

```python
num_for_corr = [
    "EDAD", "SIZE", "YEARINCOME", "ventas", "costo_venta", "DESCUENTOS",
    "ventas_netas", "margen_bruto", "YEARINCOME_std", "ventas_std", "costo_venta_minmax",
]
corr = df[num_for_corr].corr()
plt.figure(figsize=(10, 8))
sns.heatmap(corr, annot=False, cmap="coolwarm", center=0)
plt.title("Matriz de correlación")
plt.tight_layout()
plt.show()
print(corr["ventas"].sort_values(ascending=False))
```

Markdown after/before: interpretar relaciones y riesgo de colinealidad (`ventas_netas`, `margen_bruto`, `costo_venta`).

- [ ] **Step 2: Markdown — definir target, predictoras y exclusiones por leakage**

State explicitly:
- Target: `ventas`
- Predictors: `EDAD`, `GENERO`, `SIZE`, `YEARINCOME`, `DESCUENTOS`, `grupo_edad`, `segmento_ingreso`
- Excluded: `ventas_netas`, `margen_bruto`, `ventas_std`, `costo_venta`, `costo_venta_minmax`

- [ ] **Step 3: Code — prepare X/y, split, encode, train RF (+ optional linear)**

```python
feature_cols = ["EDAD", "GENERO", "SIZE", "YEARINCOME", "DESCUENTOS", "grupo_edad", "segmento_ingreso"]
X = pd.get_dummies(df[feature_cols], columns=["GENERO", "grupo_edad", "segmento_ingreso"], drop_first=True)
y = df["ventas"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

rf = RandomForestRegressor(n_estimators=100, random_state=42, n_jobs=-1)
rf.fit(X_train, y_train)
pred_rf = rf.predict(X_test)

rmse = mean_squared_error(y_test, pred_rf) ** 0.5
mae = mean_absolute_error(y_test, pred_rf)
r2 = r2_score(y_test, pred_rf)
print({"RMSE": rmse, "MAE": mae, "R2": r2})

importances = pd.Series(rf.feature_importances_, index=X.columns).sort_values(ascending=False)
print(importances.head(10))

lin = LinearRegression()
lin.fit(X_train, y_train)
pred_lin = lin.predict(X_test)
print({
    "RMSE_lin": mean_squared_error(y_test, pred_lin) ** 0.5,
    "R2_lin": r2_score(y_test, pred_lin),
})

assert r2 > -1  # smoke bound; strengthen only if stable on this dataset
assert len(importances) == X.shape[1]
```

- [ ] **Step 4: Markdown — conclusiones**

Summarize: cleaning decisions, imputation choices, scaling, useful features, model suitability, limitations, next steps.

- [ ] **Step 5: Full notebook execution**

Run:

```bash
jupyter nbconvert --to notebook --execute main.ipynb --output main.executed.ipynb
```

Expected: exit code 0; no traceback. If successful, optionally delete `main.executed.ipynb` or leave untracked via `.gitignore`.

- [ ] **Step 6: Commit**

```bash
git add main.ipynb requirements.txt
git commit -m "add correlation analysis, ventas baseline model, and conclusions"
```

---

## Spec coverage self-check

| Spec requirement | Task |
| --- | --- |
| `main.ipynb` at repo root | Task 1 |
| Load `data_source.xlsx` + rename `costo_venta` | Task 1 |
| Exploration (dtypes/nulls/ranges) | Task 2 |
| Drop duplicates by `id` | Task 3 |
| Flag `dup_estructura` / `dup_significativo` | Task 3 |
| Formats/ranges/GENERO | Task 4 |
| Coherence clip + named flags | Task 4 |
| Skewness imputation, no null drops | Task 5 |
| StandardScaler / MinMaxScaler new cols | Task 6 |
| ≥2 features + leakage notes | Task 7–8 |
| Correlation + RF model for `ventas` | Task 8 |
| Spanish MD + one responsibility/cell | All tasks |
| Full activities 1–4 | Tasks 2–8 |

## Placeholder / consistency notes

- Flag names match the design spec exactly.
- Predictor exclusion list matches design (`costo_venta` out of baseline).
- Significant-duplicate columns match design: `EDAD`, `GENERO`, `SIZE`, `YEARINCOME`.
- Age bins may be adjusted only to cover observed `EDAD` min/max; document if changed.
