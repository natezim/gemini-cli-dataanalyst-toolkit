# Analysis & ML

## pandas patterns
```python
# Merges
df = pd.merge(events, users, how="left", on="user_id")

# Missing data
missing = df.isna().mean().sort_values(ascending=False)

# Categoricals (memory + performance)
for c in ["country", "device_type"]:
    df[c] = df[c].astype("category")

# Feature engineering
df["ts"] = pd.to_datetime(df["ts"], errors="coerce")
df["hour"] = df["ts"].dt.hour
df["weekday"] = df["ts"].dt.dayofweek
```
- Many-to-many joins explode row counts — check cardinality first
- Copy-on-Write is default in pandas 3.0 — simplifies view/copy semantics
- Enforce numeric conversion early for statsmodels (object columns cause silent failures)

## scikit-learn (leakage-safe pipeline)
```python
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split, cross_validate

num_cols = X.select_dtypes(include=["number"]).columns
cat_cols = X.select_dtypes(exclude=["number"]).columns

preprocess = ColumnTransformer([
    ("num", Pipeline([("imp", SimpleImputer()), ("scl", StandardScaler())]), num_cols),
    ("cat", Pipeline([("imp", SimpleImputer(strategy="most_frequent")),
                       ("ohe", OneHotEncoder(handle_unknown="ignore"))]), cat_cols),
])

model = Pipeline([("preprocess", preprocess), ("clf", LogisticRegression(max_iter=200))])
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
model.fit(X_train, y_train)
```
Pipelines prevent leakage: fit on train, transform on test, automatically.

## statsmodels
```python
import statsmodels.api as sm
X = sm.add_constant(X)  # intercept NOT included by default
model = sm.OLS(y, X).fit()
print(model.summary())
```
Must add intercept explicitly — OLS docs call this out.

## AutoML tools (when to use which)

| Tool | Best for | Key caveat |
|------|----------|-----------|
| Optuna | Tuning a known pipeline | Needs well-designed search space |
| auto-sklearn | Strong baseline AutoML | Heavy compute |
| AutoGluon | High tabular performance | Complex artifacts |
| FLAML | Budget-constrained AutoML | Needs careful validation |
| PyCaret | Fast prototyping | Can hide leakage risks |
| H2O AutoML | Enterprise stacks | Separate runtime |

Gradient boosting libraries (often best for tabular):
- XGBoost, LightGBM, CatBoost — all have scikit-learn compatible APIs

## EDA skeleton
```python
summary = {
    "shape": df.shape,
    "dtypes": df.dtypes.astype(str).to_dict(),
    "missing": df.isna().sum().to_dict(),
    "describe": df.select_dtypes(include="number").describe().to_dict(),
}
```
