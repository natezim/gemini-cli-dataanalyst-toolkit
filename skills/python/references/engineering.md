# Engineering & Deployment

## Testing (pytest)
```python
import pytest

@pytest.fixture
def sample_data():
    return pd.DataFrame({"a": [1, 2, 3], "b": [4, 5, 6]})

def test_basic_eda(sample_data):
    result = basic_eda(sample_data, EDAConfig())
    assert "shape" in result
    assert result["shape"] == (3, 2)
```
Fixtures are pytest's core — request by name, auto-injected.

## Data validation
- **pandera**: schema validation for pandas DataFrames
- **Great Expectations**: expectation suites for data quality checkpoints
- **Pydantic**: type-hint-driven validation, emits JSON Schema (good for API boundaries)

Type hints are NOT runtime enforcement. Pair with validation libraries for real guarantees.

## Virtual environments
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```
- Pin dependencies (`pip freeze > requirements.txt` or use `pip-tools`)
- conda for when you need specific Python versions or non-Python dependencies

## Packaging
- Use `pyproject.toml` (modern standard)
- Build with `python -m build`
- For internal tools: `pip install -e .` for editable installs

## Containers
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```
- Multi-stage builds for smaller images
- Don't run as root in production

## Model serving

| Tool | Best for | Note |
|------|----------|------|
| FastAPI + Uvicorn | Custom APIs, lightweight | Most flexible |
| BentoML | ML model packaging + serving | Good abstractions |
| Ray Serve | Scalable inference | GPU scheduling |
| KServe | Kubernetes-native serving | Autoscaling, rollouts |
| TorchServe | PyTorch models | Limited maintenance (caution) |

## Experiment tracking (MLflow)
```python
import mlflow

with mlflow.start_run():
    mlflow.log_param("n_estimators", 100)
    mlflow.log_metric("f1", 0.87)
    mlflow.sklearn.log_model(model, "model")
```
MLflow Model Registry for lifecycle management (staging → production).

## Security reminders
- eval()/exec() are NEVER safe with untrusted input
- pickle is NOT secure — never unpickle untrusted data
- Treat all dynamically generated code as untrusted
- Use process/container isolation for execution, not language-level restrictions
