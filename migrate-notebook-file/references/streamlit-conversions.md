# Streamlit to Jupyter Conversion Reference

## Text Output
```python
# Before
st.write("Hello World")
st.markdown("## Header")
st.text("Plain text")
st.title("Page Title")
st.header("Section Header")
st.subheader("Subsection")
st.caption("Small caption text")
st.code("print('hello')", language="python")

# After
print("Hello World")
print("## Header")
print("Plain text")
print("# Page Title")
print("## Section Header")
print("### Subsection")
print("Small caption text")
print("print('hello')")
```

## DataFrames
```python
# Before
st.dataframe(df)
st.table(df)

# After
display(df)  # In Jupyter environments
# or for large DataFrames
df.head(20)
```

## Charts - Line Charts
```python
# Before
st.line_chart(df)
st.line_chart(df, x="date", y="value")

# After
import plotly.express as px

# Simple case (df index as x, all columns as y)
fig = px.line(df)
fig.show()

# With explicit x and y
fig = px.line(df, x="date", y="value")
fig.show()

# Multiple y columns
fig = px.line(df, x="date", y=["value1", "value2"])
fig.show()
```

## Charts - Bar Charts
```python
# Before
st.bar_chart(df)
st.bar_chart(df, x="category", y="count")

# After
import plotly.express as px

# Simple case
fig = px.bar(df)
fig.show()

# With explicit x and y
fig = px.bar(df, x="category", y="count")
fig.show()

# Grouped bar chart
fig = px.bar(df, x="category", y="count", color="group", barmode="group")
fig.show()
```

## Charts - Area Charts
```python
# Before
st.area_chart(df)

# After
import plotly.express as px

fig = px.area(df)
fig.show()

# With explicit columns
fig = px.area(df, x="date", y="value")
fig.show()
```

## Charts - Scatter Charts
```python
# Before
st.scatter_chart(df, x="x_col", y="y_col")

# After
import plotly.express as px

fig = px.scatter(df, x="x_col", y="y_col")
fig.show()

# With size and color
fig = px.scatter(df, x="x_col", y="y_col", size="size_col", color="category")
fig.show()
```

## Charts - Plotly (already using plotly)
```python
# Before
st.plotly_chart(fig)

# After
fig.show()
```

## Metrics
```python
# Before
st.metric("Revenue", "$1,000", "+10%")
st.metric(label="Users", value=1500, delta=50)

# After
print(f"Revenue: $1,000 (+10%)")
print(f"Users: 1500 (Δ +50)")
```

## Containers and Layout
```python
# Before
with st.sidebar:
    st.write("Sidebar content")

col1, col2 = st.columns(2)
with col1:
    st.write("Left")
with col2:
    st.write("Right")

with st.expander("Details"):
    st.write("Hidden content")

with st.container():
    st.write("Grouped content")

# After - Remove layout wrappers, keep content sequential
print("Sidebar content")

print("Left")
print("Right")

print("Details:")
print("Hidden content")

print("Grouped content")
```

## Caching Decorators
```python
# Before
@st.cache_data
def load_data():
    return pd.read_csv("data.csv")

@st.cache_resource
def get_connection():
    return create_engine("...")

@st.experimental_memo  # Deprecated alias
def cached_func():
    pass

@st.experimental_singleton  # Deprecated alias
def singleton_func():
    pass

# After - Use functools.lru_cache or remove caching
from functools import lru_cache

@lru_cache(maxsize=128)
def load_data():
    return pd.read_csv("data.csv")

# For resources that shouldn't be cached per-call, use module-level
_connection = None
def get_connection():
    global _connection
    if _connection is None:
        _connection = create_engine("...")
    return _connection

# Or simply remove decorator if caching not critical
def cached_func():
    pass
```

## Rerun and Stop
```python
# Before
st.rerun()
st.experimental_rerun()
st.stop()

# After - Remove or replace with control flow
# st.rerun() - Remove entirely (no equivalent in Jupyter)
# st.stop() - Replace with return or raise
raise SystemExit("Stopping execution")
# Or use return in functions
```

## Interactive Widgets (No Direct Equivalent)
These widgets have no Jupyter equivalent. **Prompt the user** to choose how to handle each widget:

**For input widgets** (`st.text_input`, `st.number_input`, `st.selectbox`, `st.slider`, etc.):
Ask the user: "How should I handle this input widget?"
- **Hardcode default value**: Use the widget's default value as a constant
- **Use editable variable**: Create a clearly-named variable the user can manually edit

**For button widgets** (`st.button`):
Ask the user: "How should I handle this button?"
- **Run unconditionally**: Remove the if-block and always execute the code
- **Use boolean flag**: Create a flag variable user sets to True to execute
- **Move to separate cell**: Put the action in its own cell for manual execution control
- **Comment out**: Disable the code block entirely

### Conversion Examples

**Input widgets:**
```python
# Before
name = st.text_input("Enter name", "default")
option = st.selectbox("Choose", ["A", "B", "C"])

# After (hardcode default)
name = "default"
option = "A"

# After (editable variable)
NAME = "default"  # Edit this value as needed
OPTION = "A"  # Edit this value as needed
```

**Button widgets:**
```python
# Before
if st.button("Submit"):
    process()

# After (run unconditionally)
process()

# After (boolean flag)
RUN_PROCESS = False  # Set to True to execute
if RUN_PROCESS:
    process()

# After (separate cell) - move to its own cell:
# --- Cell N ---
process()

# After (comment out)
# MIGRATION NOTE: Button removed - uncomment to run
# process()
```

## Status and Progress
```python
# Before
st.success("Operation completed!")
st.error("Something went wrong")
st.warning("Be careful")
st.info("FYI...")
with st.spinner("Loading..."):
    do_work()
progress = st.progress(0)
for i in range(100):
    progress.progress(i + 1)

# After
print("✓ Operation completed!")
print("✗ Something went wrong")
print("⚠ Be careful")
print("ℹ FYI...")
do_work()
from tqdm import tqdm
for i in tqdm(range(100)):
    pass  # tqdm shows progress bar in Jupyter
```

## JSON and Data Display
```python
# Before
st.json({"key": "value"})

# After
import json
print(json.dumps({"key": "value"}, indent=2))
# or in Jupyter
from IPython.display import JSON
JSON({"key": "value"})
```

## Session State
```python
# Before
if "counter" not in st.session_state:
    st.session_state.counter = 0
st.session_state.counter += 1

# After - Use regular Python variables (state resets each run)
counter = 0
counter += 1
# Or use global dict if state needed across cells
_state = {}
if "counter" not in _state:
    _state["counter"] = 0
_state["counter"] += 1
```

## File Uploaders
```python
# Before
uploaded = st.file_uploader("Choose file")
if uploaded:
    df = pd.read_csv(uploaded)

# After - Use file path directly (must be in /tmp folder)
file_path = "/tmp/file.csv"  # User must update path - all files must be in /tmp
df = pd.read_csv(file_path)
```

## Forms
```python
# Before
with st.form("my_form"):
    name = st.text_input("Name")
    submitted = st.form_submit_button("Submit")
    if submitted:
        process(name)

# After - Remove form wrapper, handle inputs directly
NAME = ""  # Edit this value
process(NAME)
```

## Tabs
```python
# Before
tab1, tab2 = st.tabs(["Tab 1", "Tab 2"])
with tab1:
    st.write("Content 1")
with tab2:
    st.write("Content 2")

# After - Sequential sections with headers
print("## Tab 1")
print("Content 1")

print("## Tab 2")
print("Content 2")
```

## Images
```python
# Before
st.image("path/to/image.png", caption="My Image")

# After
from IPython.display import Image, display
display(Image(filename="path/to/image.png"))
print("My Image")
```
