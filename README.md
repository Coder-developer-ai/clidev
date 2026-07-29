# clidev

**clidev** is a Python framework for building **production-ready Command Line
Applications (CLI)** and **Terminal User Interfaces (TUI)** with minimal code.

Unlike traditional CLI libraries that only parse arguments, `clidev` gives you
a complete framework for interactive terminal apps — menus, forms, workflows,
dashboards, state management, command execution, navigation, and
event-driven logic — so you don't have to hand-write input loops, menu
rendering, or terminal state machines.

> Think of it as the Flutter/React of terminal applications: you describe
> what your app looks like and does, `clidev` handles the rendering loop.

Built on top of [`rich`](https://github.com/Textualize/rich) for terminal
rendering and [`questionary`](https://github.com/tmbo/questionary) for
interactive prompts.

---

## Installation

```bash
pip install -e .
```

(This installs `clidev` from this repo in editable mode, along with its
dependencies: `rich`, `questionary`, `pydantic`, `click`, `PyYAML`, `toml`.)

## Quick start

```python
from clidev import App

app = App("My App")

home = app.menu("Home")
home.option("Say Hello", lambda: app.success("Hello!"))
home.option("Exit", app.exit)

app.run()
```

Or scaffold a brand-new project with the bundled CLI:

```bash
clidev new myproject
cd myproject
python app.py
```

---

## Module guide

### `App` (`clidev/app.py`)

The core object. Wires together state, storage, routing, events, plugins,
theming, and every UI widget factory.

```python
from clidev import App

app = App("Developer Toolkit", theme="dark", storage_backend="json")
```

### State (`clidev/state.py`)

Global, dict-like state, accessible anywhere in your app.

```python
app.state["username"] = "Vrushabh"
print(app.state["username"])

app.state.on_change(lambda key, old, new: print(f"{key}: {old} -> {new}"))
```

### Storage (`clidev/storage.py`)

Every form (or any code) can persist data through a pluggable backend:
`memory`, `json`, `sqlite`, `yaml`, or `toml`.

```python
app.storage.save("user", {"name": "Bob"})
user = app.storage.load("user")
```

```python
app = App("My App", storage_backend="sqlite")
```

### Forms (`clidev/forms.py`, `inputs.py`, `validators.py`)

Chainable form builder with automatic validation.

```python
form = app.form("User")
form.text("Name")
form.email("Email")
form.password("Password")
form.number("Age")

data = form.run()
# {"Name": "Vrushabh", "Email": "abc@gmail.com", "Password": "******", "Age": 17}
```

Supported field types: `text`, `email`, `password`, `number`, `url`, `file`,
`folder`, `date`, `time`, `checkbox`, `toggle`, `radio`, `select`/`dropdown`,
`multiselect`, `searchable`.

Custom validators can be attached per-field via `extra_validators=[...]`
using anything from `clidev.validators` (`min_length`, `max_length`,
`min_value`, `max_value`, `is_date`, etc.).

### Menus (`clidev/menus.py`)

```python
menu = app.menu("Main Menu")
menu.option("Create Project", create_project)
menu.option("Deploy", deploy)
menu.option("Exit", app.exit)
```

Nested menus:

```python
main = app.menu("Main")
settings = app.menu("Settings")
main.link("Settings", settings)
```

### Page routing (`clidev/router.py`, `pages.py`)

```python
@app.page("home")
def home():
    ...

app.goto("settings")
app.back()
```

### Conditional navigation (`clidev/actions.py`)

```python
@app.when(lambda data: data["Role"] == "Admin")
def admin():
    app.goto("admin_menu")
```

or:

```python
app.if_value("Role", equals="Admin").goto("admin_menu")
app.if_value("Age", greater_than=18).goto("adult_menu")
```

Supported comparisons: `equals`, `not_equals`, `greater_than`, `less_than`,
`greater_equal`, `less_equal`, `contains`, `in_list`.

### Workflow engine (`clidev/workflow.py`)

```python
workflow = app.workflow()
workflow.step(login)
workflow.step(select_project)
workflow.step(build)
workflow.step(deploy)
result = workflow.start()
```

Each step can accept a shared `context` dict; whatever a step returns (as a
dict) is merged into that context for subsequent steps.

### Events (`clidev/events.py`)

```python
@app.on_start
def startup():
    ...

@app.on_exit
def shutdown():
    ...

@app.on_submit(some_form)
def save(data):
    ...

@app.on_error
def on_error(e):
    ...

``
### Images & Video (`clidev/image.py`)

Render static images and play video clips directly inside the terminal,
using colored half-block characters (real truecolor, not ASCII-art
approximation) for near photo-quality output in supporting terminals.

```python
app.image("photo.png").show()
```

```python
app.video("clip.mp4").play()
```

**Requires optional dependencies:**
```bash
pip install clidev[media]
```
or individually:
```bash
pip install Pillow            # for images
pip install opencv-python     # for video
```
If these aren't installed, calling `.image()` or `.video()` raises a
clear `ClidevError` telling you what to install — the rest of `clidev`
works fine without them.

#### Images

```python
app.image("photo.png").show()
app.image("photo.png", width=60).show()   # control render width in columns
```

| Argument | Default | Description |
|---|---|---|
| `path` | — | Path to the image file |
| `width` | terminal width (max 120) | Render width in terminal columns |

#### Video

```python
app.video("clip.mp4").play()
app.video("clip.mp4", width=60, fps=10).play(max_frames=200)
```

| Argument | Default | Description |
|---|---|---|
| `path` | — | Path to the video file |
| `width` | terminal width (max 100) | Render width in terminal columns |
| `fps` | source video's FPS | Override playback frame rate |

`.play()` options:
- `max_frames` — stop after N rendered frames (default: play whole video)
- `skip` — render every Nth source frame, useful to keep pace on longer
  or higher-fps clips where terminal redraw can't keep up

**Terminal compatibility:** works best in terminals with truecolor support
(Windows Terminal, iTerm2, most modern Linux terminals). Classic `cmd.exe`
may render duller or incorrect colors.


### Command execution (`clidev/shell.py`)

```python
app.cmd("git init")

result = app.cmd("git status", capture=True)
print(result.stdout, result.ok)

app.cmd("pip install -r requirements.txt", background=True)
```

### Progress & tasks (`clidev/progress.py`, `spinner.py`, `tasks.py`, `scheduler.py`)

```python
with app.progress("Installing"):
    app.cmd("pip install numpy")
    app.cmd("pip install pandas")

@app.task
def build():
    ...

app.run_task("build")
```

### Plugins (`clidev/plugins.py`)

```python
from clidev.plugins import Plugin

class GitPlugin(Plugin):
    name = "git"

    def on_install(self, app):
        ...

    def on_start(self, app):
        ...

app.use(GitPlugin())
```

### Themes (`clidev/themes.py`, `colors.py`)

```python
app = App("My App", theme="dark")

theme = Theme()
theme.primary("blue")
theme.success("green")
```
### Banner (`clidev/banner.py`)

Render text as large ASCII-art block letters — great for app splash
screens, section headers, or command output.

```python
app.banner("HI").show()
```

Customize the symbol used to draw the letters:

```python
app.banner("DEPLOYED", symbol="@").show()
```

Or use it standalone (outside an `App` instance):

```python
from clidev.banner import Banner

Banner("HELLO", symbol="*").show()
```

**Supported characters:** `A-Z` (case-insensitive), `0-9`, and basic
punctuation (`. , ! ? - :`). Unsupported characters render as blank space
rather than raising an error.

**Constructor options:**

| Argument | Default | Description |
|---|---|---|
| `text` | — | Text to render (auto-uppercased) |
| `symbol` | `"#"` | Character used to draw filled pixels |
| `spacing` | `1` | Blank columns between letters |
| `theme` | `None` | If set (e.g. via `app.banner(...)`), colors the banner using the theme's primary color |

`.render()` returns the raw multi-line string (useful if you want to embed
it inside a `Card`, log it, or write it to a file) — `.show()` prints it
directly to the console with theme styling applied.

from clidev import App

app = App("My App")
app.banner("MY APP").show()
...

### Logging (`clidev/logger.py`)

```python
app.logger.info("Started")
app.logger.warning("Warning")
app.logger.error("Failed")
```

### UI widgets

- `app.table(title, columns=[...])` — data tables (`clidev/table.py`)
- `app.tree(label)` — tree views (`clidev/tree.py`)
- `app.card(title, content)` — bordered content cards (`clidev/cards.py`)
- `app.dashboard(title)` — multi-panel grid overview (`clidev/dashboard.py`)
- `app.dialog.confirm(...)`, `app.dialog.prompt(...)`, `app.dialog.alert(...)`
  (`clidev/dialogs.py`)
- `app.notify.success/error/warning/info(...)` (`clidev/notifications.py`)
- `app.statusbar.set(...).render()` (`clidev/statusbar.py`)

### Project generator (`clidev/generators/`, `cli.py`)

```bash
clidev new myproject
```

Creates:

```text
myproject/
│
├── app.py
├── routes.py
├── menus.py
├── forms.py
├── workflows.py
├── commands.py
├── storage.py
├── settings.py
└── assets/
```

---

## Full example

See [`examples/basic_app.py`](examples/basic_app.py) for the complete
"Developer Toolkit" example (menus, page routing, forms, storage, conditional
navigation, and shell commands), and
[`examples/workflow_app.py`](examples/workflow_app.py) for a workflow +
plugin + dashboard example.

```python
from clidev import App

app = App("Developer Toolkit")

home = app.menu("Home")
home.option("Create Project", "project_form")
home.option("Settings", "settings")
home.option("Exit", app.exit)


@app.page("project_form")
def project_form_page():
    project = app.form("Project")
    project.text("Project Name")
    project.select("Language", ["Python", "Rust", "Go"])
    data = project.run()

    if not data:
        return

    app.storage.save("project", data)
    app._last_form_data = data

    if data["Language"] == "Python":
        app.goto("python_setup")
    else:
        app.success(f"Project '{data['Project Name']}' created ({data['Language']}).")


@app.page("python_setup")
def python_setup():
    with app.progress("Setting up Python project"):
        app.cmd("python -m venv .venv_demo")
        app.cmd("git init")
    app.success("Project Created")


if __name__ == "__main__":
    app.run()
```

---

## Running the tests

```bash
pip install -e ".[dev]"
pytest tests/ -v
```

The test suite (in `tests/`) covers global state, all storage backends, form
field validation, the shell command wrapper, routing/history, the workflow
engine, conditional navigation, the event dispatcher, theming, the project
generator, and full `App` integration — 85 tests in total.

## Roadmap

- Additional storage backends: PostgreSQL, MongoDB, Redis
- `clidev-auth`, `clidev-cloud`, `clidev-testing`, `clidev-plugins` as
  separate installable packages
- Richer dashboard layout options and live-updating widgets

## License

MIT
