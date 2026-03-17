---
name: migrate-notebook-file
description: "Edit .ipynb notebook files to work within Notebooks in Workspaces. Use when: edit notebook for workspaces, remove streamlit from ipynb, convert notebook cells, transform notebook code."
---

# Edit Notebook Files for Workspaces Compatibility

## Overview
This skill handles direct editing of `.ipynb` notebook files to make them compatible with Notebooks in Workspaces. It focuses on code transformations within notebook cells.

## Critical Rules
- **ONLY edit `.ipynb` notebook files** - do NOT modify standalone Streamlit applications (`.py` files)
- This skill is specifically for removing Streamlit from notebooks, NOT from separate Streamlit apps
- **Cell references CANNOT be directly migrated** - legacy `cell_name.to_df()` syntax must be manually updated after execution in Workspaces

## Workflow

### Step 0: Determine Operating Environment
**Before starting any migration, determine where you are operating:**

1. **Check if running in Snowsight Workspaces** (browser-based IDE with notebook service)
2. **Check if running locally** (Cortex Code desktop, VS Code, etc.)

This affects how cell references can be handled (see Step 6).

### Step 1: Check and Add Missing Imports
**CRITICAL: Old Snowflake notebooks had certain functions available globally that MUST be explicitly imported in Workspaces.**

Scan all notebook cells for function calls and verify their imports exist. Key functions that need imports:

| Function | Required Import |
|----------|-----------------|
| `get_active_session()` | `from snowflake.snowpark.context import get_active_session` |
| `display()` | Built-in in Jupyter, but add `from IPython.display import display` if missing |
| `root` (Logging) | `from snowflake.snowpark import logging; root = logging.getLogger()` |

**Workflow:**
1. **Scan for function usage** - Look for calls to `get_active_session()`, `display()`, `root.info()`, etc.
2. **Check existing imports** - Search import cells for the required import statements
3. **Add missing imports** - If a function is used but not imported, add the import to the first code cell or create a new imports cell

**Example - `get_active_session()` migration:**
```python
# OLD (worked in legacy notebooks - globally available):
session = get_active_session()

# NEW (required in Workspaces):
from snowflake.snowpark.context import get_active_session
session = get_active_session()
```

**Common patterns to detect:**
- `session = get_active_session()` → needs `from snowflake.snowpark.context import get_active_session`
- `get_active_session().sql(...)` → same import needed
- `root.info(...)` or `root.warning(...)` → needs logging import
- `display(df)` → usually works, but add IPython import if errors occur

### Step 2: Set Explicit Session Context
Review notebook cells for references to session context that may have been implicitly set. Add explicit setup at the start of notebooks:

```python
session.use_database("DATABASE_NAME")
session.use_schema("SCHEMA_NAME")
session.use_warehouse("WAREHOUSE_NAME")
session.use_role("ROLE_NAME")  # If role was specified
```

Look for patterns that assume context is already set:
- `session.table("TABLE_NAME")` without fully qualified name
- `session.sql("SELECT * FROM TABLE_NAME")` with unqualified references
- `session.use_schema()` calls that reference variables defined elsewhere

Always place session creation and context setting before any SQL cells or use of session, preferably in the first cell of the notebook.

If context values cannot be determined, prompt the user for:
- Database name
- Schema name
- Warehouse name
- Role name (optional)

### Step 3: Preserve SQL Cells
SQL cells should be left unchanged. They are compatible with Notebooks in Workspaces as-is. Identify SQL cells by:
- `%%sql` magic command at the start
- Native SQL cell type in notebook metadata
- Cells containing only SQL statements (SELECT, CREATE, INSERT, etc.) with no Python code

### Step 4: Remove Streamlit Dependencies
For each notebook file:
1. Remove `import streamlit` and `import streamlit as st` statements
2. Convert Streamlit UI components to Jupyter equivalents (see `references/streamlit-conversions.md`)
3. Remove `%pip install streamlit` or similar magic commands from notebook cells
4. Add required imports (plotly) at the top of the notebook
5. Remove or convert caching decorators (`@st.cache_data`, `@st.cache_resource`)

### Step 5: Verify Conversion
After editing:
1. Check that no `st.` references remain: search for `st\.` pattern
2. Verify all required imports are present (plotly.express if charts were converted)
3. Run the notebook to confirm no import errors or runtime failures
4. Visually confirm outputs display correctly

### Step 6: Migrate Cell References
**CRITICAL: Cell reference syntax is INCOMPATIBLE between legacy and Workspaces notebooks.**

#### Legacy Notebook Cell References:
```python
# SQL cell named "cell1" → access in Python:
snowpark_df = cell1.to_df()
pandas_df = cell1.to_pandas()

# Reference in SQL:
SELECT * FROM {{cell1}} WHERE price > 100
```

#### Workspaces Cell References:
```python
# SQL cells auto-generate dataframe_1, dataframe_2, etc.
# These are pandas DataFrame pointers, NOT cell references

# Access in Python (already a pandas DataFrame):
df = dataframe_1  # Direct access, no .to_df() or .to_pandas()

# Reference in SQL:
SELECT * FROM {{dataframe_1}} WHERE price > 100
```

#### Key Differences:
| Aspect | Legacy Notebooks | Notebooks in Workspaces |
|--------|------------------|------------------------|
| **Naming** | User-assigned cell names (`cell1`, `my_query`) | Auto-generated `dataframe_x` names |
| **SQL → Python** | `cell_name.to_df()` or `cell_name.to_pandas()` | `dataframe_x` is already a pandas DataFrame |
| **Python → SQL** | `{{variable}}` | `{{variable}}` (same) |
| **SQL → SQL** | `{{cell_name}}` | `{{dataframe_x}}` |
| **Discovery** | Check the cell name you assigned | Hover over result tooltip to see `dataframe_x` name |

#### Migration Workflow:

**1. Scan for legacy cell references:**
Search notebook for patterns:
- `<identifier>.to_df()` - legacy SQL cell to Snowpark DataFrame
- `<identifier>.to_pandas()` - legacy SQL cell to pandas DataFrame
- `{{<cell_name>}}` in SQL cells - references to named cells

**2. If legacy cell references are found:**

**Option A: Running in Snowsight Workspaces (can execute cells)**
1. Ask user for permission to run SQL cells to discover the auto-generated `dataframe_x` names
2. Execute each SQL cell that is referenced
3. Hover/inspect the output to get the `dataframe_x` identifier
4. Update Python code:
   ```python
   # Before (legacy)
   df = my_query_cell.to_pandas()
   
   # After (workspaces) - dataframe_1 discovered from execution
   df = dataframe_1  # Already pandas, no conversion needed
   ```
5. Update SQL references:
   ```sql
   -- Before
   SELECT * FROM {{my_query_cell}}
   
   -- After
   SELECT * FROM {{dataframe_1}}
   ```

**Option B: Running locally (cannot execute in Workspaces)**
1. **Prompt the user** with this message:
   > "I found legacy cell references that cannot be directly migrated:
   > - `<list of references found>`
   > 
   > In Workspaces, SQL cell results use auto-generated names like `dataframe_1`, `dataframe_2`, etc.
   > The exact names depend on cell execution order and can only be determined at runtime.
   > 
   > **Options:**
   > 1. **Add TODO comments** - Mark each reference with a comment explaining the manual update needed
   > 2. **Use temporary placeholders** - Replace with `dataframe_N` placeholders that user updates after first run
   > 3. **Skip cell reference migration** - Leave references as-is for user to handle manually"

2. If user chooses TODO comments:
   ```python
   # TODO: MIGRATION - Update cell reference after running in Workspaces
   # Legacy: df = my_query_cell.to_pandas()
   # Workspaces: Hover over SQL cell result to find dataframe_x name, then use:
   # df = dataframe_x  # (no .to_pandas() needed - already pandas)
   df = my_query_cell.to_pandas()  # REPLACE WITH: dataframe_x
   ```

3. If user chooses placeholders:
   ```python
   # MIGRATION: Replace dataframe_N with actual name from SQL cell output
   df = dataframe_1  # Was: my_query_cell.to_pandas()
   ```

**3. Remove unnecessary conversion methods:**
In Workspaces, SQL cell results are already pandas DataFrames, so:
- `.to_pandas()` calls should be removed
- `.to_df()` calls should be removed (or replaced with `session.create_dataframe()` if Snowpark DataFrame needed)

```python
# Before (legacy) - cell result needed conversion
df = query_cell.to_pandas()

# After (workspaces) - already pandas
df = dataframe_1

# If Snowpark DataFrame is needed:
snowpark_df = session.create_dataframe(dataframe_1)
```

## Conversion Quick Reference

| Streamlit | Jupyter Equivalent |
|-----------|-------------------|
| `st.write()` | `print()` |
| `st.dataframe(df)` | `display(df)` |
| `st.line_chart()` | `px.line().show()` |
| `st.bar_chart()` | `px.bar().show()` |
| `st.plotly_chart(fig)` | `fig.show()` |
| `st.metric()` | `print(f"Label: value (delta)")` |
| `@st.cache_data` | `@lru_cache` or remove |
| `st.session_state` | Python dict `_state = {}` |

For complete conversion examples including interactive widgets, caching, forms, and edge cases, see **`references/streamlit-conversions.md`**.

## Edge Cases

### Interactive Features with No Equivalent
**Always prompt the user** to choose how to handle each interactive widget:
- Input widgets → hardcode default or editable variable
- Buttons → run unconditionally, boolean flag, separate cell, or comment out

Always add a comment explaining what was changed:
```python
# MIGRATION NOTE: Replaced st.selectbox with hardcoded default "option_a"
selected = "option_a"
```

## When to Apply
- User asks to "edit notebook for workspaces"
- User wants to "remove streamlit from ipynb file"
- User needs to "convert notebook cells"
- User mentions "transform notebook code for workspaces"
- User asks to convert `st.write`, `st.dataframe`, `st.line_chart`, etc.
- User wants to remove `@st.cache_data` or `@st.cache_resource`
- User asks to "migrate cell references" or "update cell_name.to_df()"
- Called by migrate-notebook skill for file-level transformations

## Important Notes
- **ONLY edit `.ipynb` notebook files** - never modify standalone Streamlit applications (`.py` files)
- Always preserve the original notebook logic and outputs
- Ensure plotly.express imports are added when converting visualizations
- Always use Plotly for interactive charts (NOT matplotlib or ipywidgets)
- Add migration comments for any removed interactive features
- **Cell references CANNOT be auto-migrated** - `cell_name.to_df()` → `dataframe_x` mapping requires runtime execution
- When running in Workspaces, request permission to execute cells to discover `dataframe_x` names
- When running locally, always prompt user for how to handle cell references
- Test that the migrated notebook runs correctly in a Jupyter environment
