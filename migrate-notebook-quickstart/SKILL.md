---
name: migrate-notebook-quickstart
description: "Migrate Snowflake quickstart guide .md files from old Streamlit-based notebooks to new Jupyter Notebooks in Workspaces. Use when: quickstart migration, notebook workspaces, jupyter conversion. Only updates documentation - delegates code changes to migrate-notebook skills."
---

# Migrate Notebook Quickstart

## Overview
This skill guides the migration of Snowflake quickstart **documentation (.md files only)** from old Streamlit-based notebook format to the new Jupyter Notebooks in Workspaces format.

**IMPORTANT: This skill only updates markdown documentation files. REQUIRED: invoke these skills for any related repos and notebook files:**
- Use `migrate-notebook` skill for repository-level notebook migrations
- Use `migrate-notebook-file` skill for individual .ipynb file migrations

## Core Pattern Change

### Interactive Notebooks (No Scheduling)

**Before (Old Pattern):**
```
1. Create API integration
2. Create git repo stage
3. Create notebook from git repo stage
4. Open notebook in Snowsight Notebooks tab
```

**After (New Pattern):**
```
1. Create API integration
2. Instruct user to clone repo into Workspaces
3. Create service on compute pool
```

### Scheduled Notebooks

**Before (Old Pattern):**
```
1. Create API integration
2. Create git repo stage
3. Create notebook from git repo stage
4. Create task to execute notebook on schedule
```

**After (New Pattern):**
```
1. Create API integration
2. Instruct user to clone repo into Workspaces (for development/editing)
3. Create NOTEBOOK PROJECT from git repo (for scheduling)
4. Create task to execute notebook project on schedule
```

## Workflow

### Step 1: Analyze Current Quickstart
1. Read the existing quickstart markdown file
2. Identify the notebook creation pattern - look for variations of:
   - `CREATE NOTEBOOK ... FROM '@<stage>...'`
   - `CREATE GIT REPOSITORY` or git stage references
   - "Open Notebook" or "Notebooks tab" instructions
   - `!pip install` environment setup
   - Screenshots of old notebook UI

3. **Check for scheduling** - look for:
   - `CREATE TASK` that references the notebook
   - `EXECUTE NOTEBOOK` in task definitions
   - `CALL SYSTEM$SEND_SNOWFLAKE_NOTIFICATION` with notebook
   - Schedule-related instructions (cron, intervals)
   - References to automated/scheduled execution

**Note:** Quickstarts vary significantly in structure. Focus on identifying the core pattern rather than expecting exact matches.

### Step 2: Determine Migration Path

**ALL quickstarts get Workspaces instructions** for interactive development/editing.

**If scheduling IS ALSO present:**
→ Additionally include **Notebook Project** setup (Step 2A)

### Step 2A: Notebook Project Setup (For Scheduled Notebooks)

When the quickstart schedules notebook execution, add `CREATE NOTEBOOK PROJECT` for the scheduling component:

**Before:**
```sql
-- Old pattern with scheduling
CREATE GIT REPOSITORY my_repo
   ORIGIN = 'https://github.com/...'
   API_INTEGRATION = my_integration;

CREATE NOTEBOOK my_notebook
   FROM '@my_repo/branches/main/'
   MAIN_FILE = 'notebook.ipynb'
   QUERY_WAREHOUSE = my_wh;

CREATE TASK run_notebook_task
   WAREHOUSE = my_wh
   SCHEDULE = 'USING CRON 0 9 * * * UTC'
AS
   EXECUTE NOTEBOOK my_notebook;
```

**After:**
```sql
-- New pattern with notebook project for scheduling
CREATE NOTEBOOK PROJECT my_notebook_project
   FROM 'https://github.com/...'
   MAIN_FILE = 'notebook.ipynb'
   QUERY_WAREHOUSE = my_wh
   API_INTEGRATION = my_integration;

CREATE TASK run_notebook_task
   WAREHOUSE = my_wh
   SCHEDULE = 'USING CRON 0 9 * * * UTC'
AS
   EXECUTE NOTEBOOK PROJECT my_notebook_project;
```

**Key changes for Notebook Project:**
- Use `CREATE NOTEBOOK PROJECT` instead of `CREATE NOTEBOOK`
- Reference git URL directly (no separate git repo stage needed for this purpose)
- Use `EXECUTE NOTEBOOK PROJECT` in tasks instead of `EXECUTE NOTEBOOK`
- Git repo stage may still be needed if used for other purposes (data, scripts)

**Update task instructions** to reference notebook project:
```markdown
### Schedule the Notebook
Create a notebook project and task to run it on a schedule:

` ` `sql
CREATE NOTEBOOK PROJECT <notebook_project_name>
   FROM '<git_url>'
   MAIN_FILE = '<notebook_file>'
   QUERY_WAREHOUSE = <WH>
   API_INTEGRATION = <integration>;

CREATE TASK run_notebook_task
   WAREHOUSE = <WH>
   SCHEDULE = 'USING CRON 0 9 * * * UTC'
AS
   EXECUTE NOTEBOOK PROJECT <notebook_project_name>;

ALTER TASK run_notebook_task RESUME;
` ` `
```

### Step 2B: Workspaces Instructions (For ALL Notebooks)

**ALL quickstarts should include Workspaces instructions** for interactive development and editing:

```markdown
### Create workspace
To develop and edit the notebook interactively, navigate to the Workspaces tab in Snowsight to create a git based workspace! 

From the workspace drop down, choose create from Git repository:

![git-workspace1](assets/workspace_setup1.png)

Create the workspace based on the repository url <REPO_URL>

![git-workspace2](assets/workspace_setup2.png)

Next, create a service to run the notebook on the compute pool:

![create-service1](assets/create_service1.png)

![create-service2](assets/create_service2.png)

Be sure to run this with the appropriate role and warehouse!
```

**For scheduled notebooks, add context:**
```markdown
### Create workspace
To develop, edit, and test the notebook interactively before scheduling, navigate to the Workspaces tab in Snowsight to create a git based workspace!
...
```

### Step 3: Locate the Code Repository
Check if the quickstart references an external code repository:
1. Search for GitHub URLs (e.g., `https://github.com/Snowflake-Labs/...`)
2. Look for `ORIGIN = 'https://github.com/...'` in SQL code blocks
3. Check for "GitHub Repo" links

**If NO code repository is found:**
Prompt: "I couldn't find a linked code repository in this quickstart. Please provide:
1. The path to the code repository (local or GitHub URL)
2. Or confirm that this quickstart has no associated notebook code"

### Step 4: Analyze Git Repo Stage Usage
**CRITICAL: Do not automatically remove git repo stage creation.**

Check if the git repo stage is used for purposes OTHER than notebook creation:
- Loading data files from the repo
- Referencing scripts or SQL files
- Accessing configuration files
- Any `@<repo_stage>/...` references besides `CREATE NOTEBOOK`

**Decision logic:**
- If git repo stage is ONLY used for `CREATE NOTEBOOK` → Remove the git repo stage creation
- If git repo stage is used for OTHER purposes → Keep the git repo stage, only remove the `CREATE NOTEBOOK` statement

**Example - KEEP git repo stage:**
```sql
-- Keep this if repo is used for data/scripts
CREATE GIT REPOSITORY my_repo ...;

-- Used for data loading (KEEP)
COPY INTO my_table FROM @my_repo/branches/main/data/;

-- Used for notebook (REMOVE just this part)
CREATE NOTEBOOK ... FROM '@my_repo/branches/main/';
```

**Example - REMOVE git repo stage:**
```sql
-- Remove if ONLY used for notebook creation
CREATE GIT REPOSITORY my_repo ...;
ALTER GIT REPOSITORY my_repo FETCH;
CREATE NOTEBOOK ... FROM '@my_repo/branches/main/';
-- No other @my_repo references exist
```

### Step 5: Update SQL Code Blocks

**Always REMOVE old notebook creation:**
```sql
-- Remove variations of:
CREATE OR REPLACE NOTEBOOK ...
CREATE NOTEBOOK ...
FROM '@<stage>/...'
MAIN_FILE = '...'
QUERY_WAREHOUSE = ...
RUNTIME_NAME = ...
COMPUTE_POOL = ...
```

**For scheduled notebooks, ADD notebook project:**
```sql
CREATE NOTEBOOK PROJECT <name>
   FROM '<git_url>'
   MAIN_FILE = '<notebook_file>'
   QUERY_WAREHOUSE = <wh>
   API_INTEGRATION = <integration>;
```

**Conditionally REMOVE git repo stage (only if solely used for notebooks):**
```sql
-- Remove if only used for notebook creation:
CREATE OR REPLACE GIT REPOSITORY <name> ...;
ALTER GIT REPOSITORY <name> FETCH;
```

**UPDATE API integration (if present):**
```sql
-- Add SNOWFLAKE_GITHUB_APP authentication if missing
CREATE OR REPLACE API INTEGRATION <name>
   api_provider = git_https_api
   api_allowed_prefixes = ('https://github.com/...')
   API_USER_AUTHENTICATION = (TYPE = SNOWFLAKE_GITHUB_APP)  -- ADD if missing
   enabled = true;
```

**UPDATE compute pool (if present):**
```sql
-- Consider increasing MAX_NODES for Container Runtime workloads
CREATE COMPUTE POOL IF NOT EXISTS <name>
 MIN_NODES = 1
 MAX_NODES = 3  -- Increase from 1 if needed
 INSTANCE_FAMILY = CPU_X64_M;
```

**UPDATE task definitions (for scheduled notebooks):**
```sql
-- Change EXECUTE NOTEBOOK to EXECUTE NOTEBOOK PROJECT
-- Before:
EXECUTE NOTEBOOK my_notebook;
-- After:
EXECUTE NOTEBOOK PROJECT my_notebook_project;
```

### Step 6: Remove Outdated Sections

Look for and remove or update:
- `!pip install` code blocks (packages handled by container runtime)
- "Environment Configuration" sections specific to old notebooks
- References to `SYSTEM$BASIC_RUNTIME`
- Links saying "notebook is also hosted in GitHub" (now users clone directly via Workspaces)

### Step 7: Handle Screenshots
**Ask the user:**
"The migration may require new screenshots for the Workspaces UI. Would you like me to:
1. Add placeholder image references to capture manually
2. Skip screenshots and add TODO comments
3. Keep existing screenshots and only update text"

Common screenshots needed:
- `workspace_setup1.png` - Workspace dropdown
- `workspace_setup2.png` - Git repository dialog
- `create_service1.png` - Service creation
- `create_service2.png` - Service configuration

### Step 8: Update Descriptive Text

Update any text that describes the old flow:
- "create the notebook" → "create the necessary objects"
- "Notebooks tab" → "Workspaces tab"
- "open the notebook" → "create a workspace"
- "EXECUTE NOTEBOOK" → "EXECUTE NOTEBOOK PROJECT" (for scheduled)

### Step 9: Prompt for Code Migration
After completing .md updates:

"Documentation updates complete! 

If this quickstart has associated .ipynb notebook files, those may also need migration:
- **Repository migration**: Use `migrate-notebook` skill
- **Individual file migration**: Use `migrate-notebook-file` skill

Would you like me to invoke one of these skills now?"

## Flexibility Guidelines

**Quickstarts vary significantly.** Adapt the migration based on what you find:

| If you find... | Then... |
|----------------|---------|
| Scheduling/tasks | Include BOTH Workspaces instructions AND Notebook Project setup |
| Interactive only | Include Workspaces instructions only |
| Inline SQL in markdown | Update the code blocks directly |
| References to `assets/setup.sql` | Note it needs separate migration |
| Multiple notebooks | Apply appropriate pattern to each |
| No compute pool | May not need Container Runtime instructions |
| Custom runtime config | Preserve non-notebook settings |
| Git stage used for data | Keep the git stage, remove only notebook creation |

## Scope

**IN SCOPE:**
- Quickstart .md file text and structure
- SQL code blocks within .md files
- Screenshot references
- Navigation instructions

**OUT OF SCOPE (use other skills):**
- Actual .ipynb files → `migrate-notebook-file`
- Repository-wide changes → `migrate-notebook`
- Standalone .sql files → `migrate-notebook`

## When to Apply
- Documentation contains `CREATE NOTEBOOK` pattern
- Instructions reference old "Notebooks tab" navigation
- Quickstart needs Workspaces/Container Runtime update
- Quickstart has scheduled notebook execution

## Common Pitfalls
- Don't forget Workspaces instructions even for scheduled notebooks
- Don't blindly remove git repo stage - check if it's used elsewhere
- Remember `EXECUTE NOTEBOOK` → `EXECUTE NOTEBOOK PROJECT` for tasks
- Adapt instructions to the specific quickstart's context
- Remember to prompt about screenshots
- Always prompt for code repo if not found in documentation
