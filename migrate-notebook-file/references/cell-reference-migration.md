# Cell Reference Migration: Legacy to Workspaces

## Overview
Cell references work fundamentally differently between legacy Snowflake Notebooks and Notebooks in Workspaces. This reference explains the differences and migration strategies.

## Key Differences

### Naming Convention
| Legacy | Workspaces |
|--------|------------|
| User-assigned cell names (`cell1`, `my_query`) | Auto-generated `dataframe_x` names (`dataframe_1`, `dataframe_2`) |
| Cell names persist in notebook metadata | Names generated at runtime based on execution order |

### SQL Cell Results → Python

**Legacy:**
```python
# SQL cell named "cell1" returns a reference object
snowpark_df = cell1.to_df()      # Convert to Snowpark DataFrame
pandas_df = cell1.to_pandas()    # Convert to pandas DataFrame
```

**Workspaces:**
```python
# SQL cells expose results as pandas DataFrame pointers
# Named dataframe_1, dataframe_2, etc. based on execution order

# Direct access - already a pandas DataFrame, no conversion needed
pandas_df = dataframe_1

# If you need a Snowpark DataFrame:
from snowflake.snowpark.context import get_active_session
session = get_active_session()
snowpark_df = session.create_dataframe(dataframe_1)
```

### Python Variables → SQL

**Both Legacy and Workspaces (syntax unchanged):**
```sql
-- Reference Python variable in SQL using double curly braces
SELECT * FROM {{my_table_var}} WHERE price > {{threshold}}
```

### SQL Cell → SQL Cell

**Legacy:**
```sql
-- Reference another SQL cell by its name
SELECT * FROM {{cell1}} WHERE category = 'A'
```

**Workspaces:**
```sql
-- Reference by auto-generated dataframe name
SELECT * FROM {{dataframe_1}} WHERE category = 'A'
```

### Discovering DataFrame Names in Workspaces

In Workspaces, to find the `dataframe_x` name for a SQL cell:
1. Execute the SQL cell
2. Hover over the result tooltip
3. The tooltip shows the DataFrame name (e.g., `dataframe_1`)

The naming follows execution order:
- First SQL cell executed → `dataframe_1`
- Second SQL cell executed → `dataframe_2`
- etc.

## Migration Patterns

### Pattern 1: Simple SQL → pandas
```python
# Legacy
df = my_query.to_pandas()
display(df)

# Workspaces (after discovering dataframe_1 from execution)
df = dataframe_1  # Already pandas
display(df)
```

### Pattern 2: SQL → Snowpark DataFrame for further processing
```python
# Legacy
sdf = query_cell.to_df()
result = sdf.filter(col("amount") > 100).group_by("category").count()

# Workspaces
from snowflake.snowpark.context import get_active_session
session = get_active_session()
sdf = session.create_dataframe(dataframe_1)  # Convert pandas to Snowpark
result = sdf.filter(col("amount") > 100).group_by("category").count()
```

### Pattern 3: Chained SQL references
```python
# Legacy - SQL cells reference each other by name
# Cell "base_query": SELECT * FROM sales
# Cell "filtered":   SELECT * FROM {{base_query}} WHERE region = 'US'
# Cell "aggregated": SELECT category, SUM(amount) FROM {{filtered}} GROUP BY 1

# Workspaces - use dataframe_x names
# Cell 1: SELECT * FROM sales                        → dataframe_1
# Cell 2: SELECT * FROM {{dataframe_1}} WHERE region = 'US'  → dataframe_2
# Cell 3: SELECT category, SUM(amount) FROM {{dataframe_2}} GROUP BY 1
```

### Pattern 4: Python processing between SQL cells
```python
# Legacy
raw_data = data_cell.to_pandas()
processed = raw_data.dropna()
# Then reference 'processed' in SQL: SELECT * FROM {{processed}}

# Workspaces
raw_data = dataframe_1  # From SQL cell result
processed = raw_data.dropna()
# Then reference 'processed' in SQL: SELECT * FROM {{processed}}
# (Python variable referencing works the same)
```

## Common Errors After Migration

### Error: `NameError: name 'cell1' is not defined`
**Cause:** Legacy cell reference used in Workspaces
**Fix:** Replace `cell1` with the corresponding `dataframe_x` name

### Error: `AttributeError: 'DataFrame' object has no attribute 'to_pandas'`
**Cause:** Calling `.to_pandas()` on a Workspaces result (already pandas)
**Fix:** Remove `.to_pandas()` - access the dataframe directly

### Error: `AttributeError: 'DataFrame' object has no attribute 'to_df'`
**Cause:** Calling `.to_df()` on a Workspaces result (already pandas)
**Fix:** Use `session.create_dataframe(dataframe_x)` if Snowpark DataFrame needed

## Migration Checklist

- [ ] Scan for `.to_df()` calls - replace with `session.create_dataframe(dataframe_x)`
- [ ] Scan for `.to_pandas()` calls - remove, access `dataframe_x` directly
- [ ] Scan for `{{cell_name}}` in SQL - replace with `{{dataframe_x}}`
- [ ] Execute notebook in order to establish `dataframe_x` mapping
- [ ] Update all references with discovered `dataframe_x` names
- [ ] Test notebook runs end-to-end without cell reference errors
