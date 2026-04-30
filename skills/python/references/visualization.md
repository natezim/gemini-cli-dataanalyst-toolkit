# Visualization

## Choosing the right tool

| Tool | Use for | Style |
|------|---------|-------|
| matplotlib | Publication-quality static plots, full control | Imperative (Figure/Axes) |
| seaborn | Quick EDA with sensible defaults | High-level (on matplotlib) |
| Plotly | Interactive exploration, dashboards | Declarative + Express |
| Altair | Compact specs, consistent encodings | Declarative (Vega-Lite) |
| Bokeh | Python callbacks, interactive apps | Python-first interactive |

Choose based on the analytical question, not habit. Different questions need different plots.

## Minimal idioms

**matplotlib:**
```python
fig, ax = plt.subplots()
ax.hist(df["value"].dropna(), bins=40)
ax.set(title="Value distribution", xlabel="value", ylabel="count")
```

**seaborn:**
```python
sns.histplot(data=df, x="value", hue="segment", kde=True)
```

**Plotly Express:**
```python
fig = px.scatter(df, x="x", y="y", color="segment", hover_data=["id"])
fig.show()
```

**Altair:**
```python
alt.Chart(df).mark_point().encode(
    x="x:Q", y="y:Q", color="segment:N", tooltip=["id:N", "x:Q", "y:Q"]
)
```

## Common pitfalls
- Auto-choosing chart type without checking intent (distribution? relationship? trend?)
- Rendering millions of points interactively — use aggregation or Datashader
- seaborn's hidden aggregations — understand what default stat functions do
- Plotly's large payloads on huge datasets — performance degrades
- Altair's browser data size limits — aggregate first for large datasets
- Bokeh server required for Python callbacks — adds deployment complexity
