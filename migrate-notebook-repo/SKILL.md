---
name: migrate-notebook-repo
description: "Migrate Snowflake notebook repositories to Workspaces format. Use when: migrate notebook repo, convert to workspaces, workspaces migration, notebook migration, remove streamlit, streamlit to jupyter."
---

# Migrate Snowflake Notebook solutions to Workspaces

## Overview
This skill orchestrates the migration of notebook solutions from the original Snowflake notebook format to the new Jupyter-based Notebooks in Workspaces format. It handles repository-level changes, environment file cleanup, and Snowflake object setup.

## Critical Rules
- **Do NOT create notebook project objects** (`CREATE NOTEBOOK ... FROM '@stage/...'`) unless there are existing scheduled `EXECUTE NOTEBOOK` statements or task definitions in the repository
- **Only remove Streamlit from NOTEBOOKS** (`.ipynb` files) - do NOT modify separate Streamlit applications (standalone `.py` files with Streamlit code that are NOT embedded in notebooks)
- **ALWAYS extract and apply the role** from setup scripts - the role is required for notebooks to have proper permissions

## Prerequisites
- User has a directory/repo containing `.ipynb` files
- Notebooks may use Streamlit (skill auto-detects and handles this)
- User knows target database/schema/warehouse (or will provide when prompted)

## Example Scenarios
- "I have a Streamlit notebook I need to run in Workspaces" → Full migration (Steps 1-8)
- "My scheduled notebook has CREATE NOTEBOOK statements that need updating" → Steps 6-8
- "Convert my repo's notebooks to work without Streamlit" → Steps 1-5

## Workflow

### Step 1: Identify Notebooks and Related Files
Scan the current subdirectory for:
- `.ipynb` notebook files
- Environment files: `environment.yaml`, `requirements.txt`
- Setup scripts with `CREATE NOTEBOOK` statements
- Task definitions that schedule notebook execution

### Step 2: Extract Session Context from Setup Files
Parse setup scripts thoroughly to extract session context. Look for **both** explicit parameters and implicit context from preceding statements:

**⚠️ MANDATORY: Role Detection for ALL CREATE NOTEBOOK Statements**
The role is REQUIRED for proper notebook execution. You MUST:

1. **Scan the setup script** for `USE ROLE <role>` statements that appear BEFORE `CREATE NOTEBOOK` statements
2. **Track role state sequentially** - each `USE ROLE` changes the active role for all subsequent statements
3. **Map role to each notebook** - the role active when `CREATE NOTEBOOK` was executed is the role that notebook needs

**This applies to ALL CREATE NOTEBOOK statements**, not just "old-style" ones. Even if the CREATE NOTEBOOK uses `FROM '@stage/...'` syntax, the role context is still critical.

Example - setup.sql:
```sql
USE ROLE ACCOUNTADMIN;
-- ... setup statements ...

USE ROLE ML_MODEL_HOL_USER;        -- Role is now ML_MODEL_HOL_USER

CREATE WAREHOUSE ML_HOL_WH;
CREATE DATABASE ML_HOL_DB;
-- ...

CREATE OR REPLACE NOTEBOOK ML_HOL_DB.ML_HOL_SCHEMA.MY_NOTEBOOK
FROM '@...' 
MAIN_FILE = 'notebook.ipynb'       -- This notebook needs ML_MODEL_HOL_USER role
QUERY_WAREHOUSE = ML_HOL_WH;
```

**Extracted context for notebook.ipynb:**
```
notebook.ipynb:
  - role: ML_MODEL_HOL_USER    ← From USE ROLE before CREATE NOTEBOOK
  - database: ML_HOL_DB        ← From fully qualified name
  - schema: ML_HOL_SCHEMA      ← From fully qualified name
  - warehouse: ML_HOL_WH       ← From QUERY_WAREHOUSE
```

**From `CREATE NOTEBOOK` statements:**
- `QUERY_WAREHOUSE = <warehouse>` → warehouse name
- Database and schema from fully qualified name (e.g., `CREATE NOTEBOOK DB.SCHEMA.NAME`)
- Any `COMMENT` tags that reference configuration

**From preceding session context statements:**
Scan the setup script for `USE` statements that appear BEFORE each `CREATE NOTEBOOK`:
```sql
USE ROLE <role>;           -- Extract role
USE WAREHOUSE <warehouse>; -- Extract warehouse (if not in CREATE NOTEBOOK)
USE DATABASE <database>;   -- Extract database
USE SCHEMA <schema>;       -- Extract schema
USE SCHEMA <database>.<schema>; -- Extract both
```

**CRITICAL: Role Detection for Old-Style CREATE NOTEBOOK Statements**
When an old-style `CREATE NOTEBOOK` statement is found (inline content, not `FROM '@stage/...'`), you MUST determine what role was active in the session at that point:

1. **Scan backwards** from the `CREATE NOTEBOOK` statement to find the most recent `USE ROLE <role>` statement
2. **Track role state** as you parse the setup script sequentially - each `USE ROLE` changes the active role for subsequent statements
3. **Map role to each notebook** - the role that was active when `CREATE NOTEBOOK` was executed should be set in that notebook's session setup

Example parsing:
```sql
USE ROLE DATA_ENGINEER;        -- Role is now DATA_ENGINEER
CREATE NOTEBOOK notebook1 ...;  -- notebook1 should use DATA_ENGINEER

USE ROLE ML_ENGINEER;          -- Role is now ML_ENGINEER  
CREATE NOTEBOOK notebook2 ...;  -- notebook2 should use ML_ENGINEER
CREATE NOTEBOOK notebook3 ...;  -- notebook3 should use ML_ENGINEER (still active)
```

**If role cannot be determined** (no `USE ROLE` statement found before a `CREATE NOTEBOOK`), **you MUST prompt the user**:
> "I found a CREATE NOTEBOOK statement for `<notebook_name>`, but I couldn't find a USE ROLE statement before it in the setup script. What role should this notebook use for its session?"

Do NOT skip the role or use a default - the role is critical for proper notebook execution.

**Build context map per notebook:**
For each notebook file, create a context map. **Print this map to confirm extraction before proceeding:**
```
=== Extracted Session Context ===
notebook_name.ipynb:
  - role: <from USE ROLE>           ← REQUIRED
  - database: <from USE DATABASE or fully qualified name>
  - schema: <from USE SCHEMA or fully qualified name>
  - warehouse: <from QUERY_WAREHOUSE or USE WAREHOUSE>
```

**If context values cannot be determined from setup files**, prompt the user:
- "What role should this notebook use?" (REQUIRED - do not skip)
- "What database should this notebook use?"
- "What schema should this notebook use?"
- "What warehouse should this notebook use?"

### Step 3: Edit Notebook Files
**Invoke the `migrate-notebook-file` skill** for each `.ipynb` file, passing the extracted context map.

**CRITICAL: Check for existing SQL session context cells FIRST**
Before adding any session context, scan the notebook for existing SQL cells that already set context:
```sql
-- Look for cells containing any of these patterns:
USE ROLE <role>;
USE WAREHOUSE <warehouse>;
USE DATABASE <database>;
USE SCHEMA <schema>;
```

**If SQL context cells already exist:**
1. **Do NOT add duplicate Python `session.use_*()` calls** - this creates redundant context setting
2. **Add missing context to the existing SQL cell** - e.g., if `USE ROLE` is missing, add it to the SQL cell
3. **Only add the `get_active_session` import** to the Python imports cell

Example - if notebook already has:
```sql
-- Using Warehouse, Database, and Schema created during Setup
USE WAREHOUSE ML_HOL_WH;
USE DATABASE ML_HOL_DB;
USE SCHEMA ML_HOL_SCHEMA;
```

Update it to include role (do NOT add Python session.use_* calls):
```sql
-- Using Role, Warehouse, Database, and Schema created during Setup
USE ROLE ML_MODEL_HOL_USER;
USE WAREHOUSE ML_HOL_WH;
USE DATABASE ML_HOL_DB;
USE SCHEMA ML_HOL_SCHEMA;
```

**If NO SQL context cells exist:**
Only then add Python session context after `get_active_session()`:
```python
# Session context (migrated from setup script)
from snowflake.snowpark.context import get_active_session
session = get_active_session()
session.use_role("ROLE_NAME")
session.use_warehouse("WAREHOUSE_NAME")
session.use_database("DATABASE_NAME")
session.use_schema("SCHEMA_NAME")
```

**IMPORTANT:** If the original setup script contained old-style `CREATE NOTEBOOK` statements (not using `FROM '@stage/...'`), the role MUST be included in the session setup. This role should be the one that was active in the session when the original `CREATE NOTEBOOK` was executed (determined by scanning for the most recent `USE ROLE` statement before it).

**Important:** Use the SAME values or variable references that were in the original setup scripts.

**If setup script used variable references (e.g., `${DB_NAME}`, `$WAREHOUSE`):**
For SQL cells:
```sql
USE ROLE IDENTIFIER($ROLE_NAME);
USE WAREHOUSE IDENTIFIER($WAREHOUSE_NAME);
```

For Python cells (only if no SQL context cell exists):
```python
import os
session.use_role(os.environ["ROLE_NAME"])
session.use_warehouse(os.environ["WAREHOUSE_NAME"])
```

**Additional transformations handled by migrate-notebook-file:**
- Removing `import streamlit` and all `st.*` calls
- Converting Streamlit charts to Plotly (`st.line_chart` → `px.line`, etc.)
- Handling interactive widgets (prompts user for each: hardcode, flag, or comment out)
- Adding migration comments for removed features

### Step 4: Clean Environment Files
For environment files in the repository:
1. Remove `streamlit` from `environment.yaml` dependencies
2. Remove `streamlit` from `requirements.txt`

### Step 5: Check Package Availability
Scan for all packages used in the notebooks:
1. Parse `import` and `from ... import` statements in `.ipynb` cells
2. Read `environment.yaml` dependencies
3. Read `requirements.txt` entries
4. Check `%pip install` and `!pip install` magic commands in notebook cells

**Compare against base image packages:**
The Notebooks in Workspaces container base image includes common packages (pandas, numpy, snowflake-snowpark-python, plotly, scikit-learn, etc.). For each package found:
- If package is in base image → no action needed
- If package is NOT in base image → add to "missing packages" list

**If missing packages are found, prompt the user:**
> "The following packages are not in the Workspaces base image: `[package_list]`. Would you like to create an external access integration to PyPI so notebooks can install these at runtime?"

Options:
1. **Yes, create PyPI integration** → Generate SQL (see below)
2. **No, I'll handle this manually** → Continue without integration
3. **Show me what's in the base image** → List available packages

**PyPI External Access Integration SQL:**
```sql
-- Network rule for PyPI access
CREATE OR REPLACE NETWORK RULE <db>.<schema>.pypi_network_rule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('pypi.org', 'files.pythonhosted.org');

-- External access integration
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION pypi_access_integration
  ALLOWED_NETWORK_RULES = (<db>.<schema>.pypi_network_rule)
  ENABLED = TRUE;
```

**Add to notebook setup cell:**
```python
# Install packages not in base image (requires pypi_access_integration)
%pip install <missing_packages>
```

### Step 6: Remove Old-Style CREATE NOTEBOOK SQL
Search for and remove SQL that creates old-style notebooks (inline content, not FROM stage):
- `CREATE NOTEBOOK` statements that do NOT use `FROM '@stage/...'` syntax
- `CREATE OR REPLACE NOTEBOOK` statements without `FROM` clause
- Related `ALTER NOTEBOOK` statements for old-style notebooks

**Preserve git repo stages if used elsewhere:**
Before removing any `CREATE GIT REPOSITORY` or `API INTEGRATION` statements, check if they are used for:
1. Creating notebook project objects (`CREATE NOTEBOOK ... FROM '@<git_repo>/...'`)
2. `COPY FILES` operations
3. Any other references in the codebase (tasks, procedures, scripts)

If the git repo stage has ANY use other than old-style `CREATE NOTEBOOK` statements, **keep it intact**.

### Step 7: Set Up Git Integration
Create SQL to set up git integration so users can clone the repo into their Workspace:

```sql
CREATE OR REPLACE API INTEGRATION <integration_name>
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('<repo_url_prefix>')
  ENABLED = TRUE;
```

### Step 8: Handle Scheduled Notebook Execution (If Applicable)
**CRITICAL: ONLY perform this step if the repository ALREADY contains `EXECUTE NOTEBOOK` statements or task definitions that schedule notebook execution.** Do NOT create notebook project objects for repositories that don't already have scheduling.

If the repository already schedules notebook execution via tasks, transform the flow:

1. Create git API integration (Step 7)
2. Create git stage:
```sql
CREATE OR REPLACE GIT REPOSITORY <db>.<schema>.<git_repo_name>
  API_INTEGRATION = <integration_name>
  ORIGIN = '<repo_url>';
```

3. Copy from git stage to regular stage:
```sql
COPY FILES INTO @<target_stage>/<path>/
  FROM @<git_repo_name>/branches/<branch>/<notebook_path>/;
```

4. Create notebook project object:
```sql
CREATE OR REPLACE NOTEBOOK <notebook_name>
  FROM '@<target_stage>/<path>/'
  MAIN_FILE = '<notebook_filename>.ipynb'
  QUERY_WAREHOUSE = <warehouse>;
```

5. Schedule execution via task:
```sql
CREATE OR REPLACE TASK <task_name>
  WAREHOUSE = <warehouse>
  SCHEDULE = '<schedule>'
AS
  EXECUTE NOTEBOOK <notebook_name>;
```

## When to Apply
- User asks to "migrate notebook to workspaces"
- User wants to "remove streamlit from notebook"
- User needs to "convert Snowflake notebook format"
- User mentions "notebook compatibility with workspaces"
- Repository contains notebooks with Streamlit that need to run in Workspaces

## Important Notes
- **Do NOT create notebook project objects** (`CREATE NOTEBOOK ... FROM '@stage/...'`) unless the repository ALREADY has `EXECUTE NOTEBOOK` statements or scheduled tasks
- **Do NOT create git stage** unless required for existing scheduled execution workflows
- **Only remove Streamlit from NOTEBOOKS** - separate Streamlit apps (`.py` files) should NOT be modified by this skill
- **PRESERVE existing git repo stages** if they are used for notebook project objects, COPY FILES, or any other purpose beyond old-style CREATE NOTEBOOK
- Always delegate `.ipynb` file editing to the `migrate-notebook-file` skill
- Preserve the original notebook logic and outputs
- Test that the migrated notebook runs correctly in a Jupyter environment
