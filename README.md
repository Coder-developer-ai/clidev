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

For image/video rendering (local files + URLs):
```bash
pip install -e ".[media]"
```

For audio extraction without a system ffmpeg install:
```bash
pip install -e ".[audio-extract]"
```

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

### Images & Video (`clidev/image.py`)

Render static images and play video clips directly inside the terminal,
using colored half-block characters (real truecolor, not ASCII-art
approximation) for near photo-quality output in supporting terminals.
Sources can be a local file, a direct URL, or a Google Drive share link
— and video playback can include synced audio.

```python
app.image("photo.png").show()
```

```python
app.video("clip.mp4").play()
```

**Requires optional dependencies:**
```bash
pip install clidevkit[media]
```
or individually:
```bash
pip install Pillow            # for images
pip install opencv-python     # for video
pip install requests          # for loading from a URL or Google Drive
pip install pygame            # for audio playback
```
If a dependency needed for what you're calling isn't installed,
`.image()` / `.video()` raise a clear `ClidevError` telling you exactly
what to install — the rest of `clidev` works fine without them.

#### Images

```python
app.image("photo.png").show()
app.image("photo.png", size="small").show()   # 40 columns wide
app.image("photo.png", size="medium").show()  # 80 columns wide (default)
app.image("photo.png", size="large").show()   # 120 columns wide
app.image("photo.png", width=60).show()       # explicit width overrides size
```

| Argument | Default | Description |
|---|---|---|
| `path` | — | Local file path, direct URL, or Google Drive share link |
| `size` | `"medium"` | `"small"` (40 cols), `"medium"` (80 cols), or `"large"` (120 cols) |
| `width` | `None` | Explicit width in terminal columns; overrides `size` if set |

#### Video

```python
app.video("clip.mp4").play()                        # 24 fps, silent by default
app.video("clip.mp4", audio=True).play()             # with synced sound
app.video("clip.mp4", size="small").play()           # smaller renders faster
app.video("clip.mp4", fps=30).play(max_frames=200)
```

| Argument | Default | Description |
|---|---|---|
| `path` | — | Local file path, direct URL, or Google Drive share link |
| `size` | `"medium"` | `"small"` (40 cols), `"medium"` (80 cols), or `"large"` (100 cols) |
| `width` | `None` | Explicit width in terminal columns; overrides `size` if set |
| `fps` | `24` | Target playback frame rate |
| `audio` | `False` | Extract and play the video's audio track, synced to playback |

`.play()` options:
- `max_frames` — stop after N rendered frames (default: play whole video)
- `skip` — render every Nth source frame; increase this if the source
  video's native fps is much higher than your target fps

**Playback accuracy:** frame timing is measured with a monotonic clock
and accounts for how long each frame actually took to render, so
playback holds close to your target fps instead of gradually drifting
slower the longer the clip plays. Smaller `size` values render faster
per frame, which matters if you're trying to hit a full 24fps on a
slower machine or a wide terminal.

#### Loading from a URL or Google Drive

```python
app.image("https://example.com/photo.png").show()
app.image("https://drive.google.com/file/d/FILE_ID/view?usp=sharing").show()

app.video("https://example.com/clip.mp4", audio=True).play()
app.video("https://drive.google.com/file/d/FILE_ID/view", audio=True).play()
```

Direct URLs and Google Drive share links are both detected automatically
— no different method or flag needed, just pass the link as `path`.
Files are downloaded to a temporary location, used for rendering/playback,
and cleaned up automatically afterward. Google Drive's "file too large to
scan for viruses" confirmation step for big files is handled automatically.

Requires the `requests` package (`pip install requests`).

#### Audio playback

```python
app.video("clip.mp4", audio=True).play()
```

Audio is extracted from the video's own audio track and played back in
sync, starting right alongside the first rendered frame. Extraction uses
`moviepy` if installed, falling back to a system `ffmpeg` binary on PATH
if not. If neither is available, or the source has no audio track,
playback continues silently rather than raising an error — audio is
best-effort, not a hard requirement for `.play()` to work.

```toml
# Optional, for audio extraction without a system ffmpeg install:
pip install moviepy
```

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

Plugins are reusable units of behavior that hook into an app's lifecycle
(startup, shutdown, page navigation, form submission, shell commands,
errors) without wiring that logic into every app you build. Write a
plugin once, drop it into any `clidev` app with `app.use(...)`.

#### Basic usage

```python
from clidev.plugins import Plugin

class GitPlugin(Plugin):
    name = "git"

    def on_install(self, app):
        # Fires immediately when app.use(GitPlugin()) runs
        result = app.cmd("git rev-parse --is-inside-work-tree", capture=True)
        app.state["git_repo"] = result.ok

    def on_start(self, app):
        # Fires when app.run() is called
        if app.state.get("git_repo"):
            app.logger.info("Git repository detected.")

    def on_exit(self, app):
        # Fires when the app exits
        app.logger.info("Goodbye from GitPlugin!")

app = App("My App")
app.use(GitPlugin())
app.run()
```

#### Available lifecycle hooks

Override only the ones you need — every hook is optional.

| Hook | Fires when |
|---|---|
| `on_install(app)` | Immediately, when `app.use(plugin)` runs |
| `on_start(app)` | The app starts (`app.run()`) |
| `on_page(app, page_name)` | Every page navigation (`app.goto(name)`) |
| `on_submit(app, form_title, data)` | Every form submission |
| `on_command(app, command, result)` | After every shell command (`app.cmd(...)`) |
| `on_error(app, error)` | Whenever an error is emitted through `app.events` |
| `on_exit(app)` | The app exits |

You're not limited to this list — **any method named `on_<something>`**
is automatically a valid hook. Apps and plugins can fire their own custom
events too:

```python
app.plugins.dispatch("on_deploy_finished", environment="production")
```
and any installed plugin defining `on_deploy_finished(self, app, environment)`
will receive it.

#### Ordering with `priority`

Plugins run in ascending `priority` order (lower = earlier), with
registration order breaking ties.

```python
class LoggingPlugin(Plugin):
    name = "logging"
    priority = 0   # runs before everything else

class AnalyticsPlugin(Plugin):
    name = "analytics"
    priority = 50  # runs after LoggingPlugin
```

#### Passing configuration

Plugins accept config as keyword arguments at registration time, and
read it back with `self.get(...)`:

```python
class AnalyticsPlugin(Plugin):
    name = "analytics"

    def on_start(self, app):
        app.logger.info(f"Using API key: {self.get('api_key')}")

app.use(AnalyticsPlugin(), api_key="abc123")
```

#### Declaring dependencies between plugins

```python
class AnalyticsPlugin(Plugin):
    name = "analytics"
    requires = ["auth"]   # must be installed before this plugin

app.use(AuthPlugin())
app.use(AnalyticsPlugin())   # fine, "auth" is already installed

app.use(AnalyticsPlugin())   # raises PluginError if "auth" wasn't installed first
```

#### Reusing plugins across apps with the global registry

Register a plugin class once, then any app can pull it in by name
instead of importing the class directly:

```python
app.plugins.register_class("git", GitPlugin)
app.plugins.register_class("analytics", AnalyticsPlugin)

app.use("git")
app.use("analytics", api_key="abc123")

app.plugins.available()   # ["git", "analytics"]
```

#### Quick one-off plugins without a class

```python
from clidev.plugins import FunctionalPlugin

def greet(app):
    app.logger.info("Hello from a functional plugin!")

app.use(FunctionalPlugin("greeter", on_start=greet))
```

#### Managing installed plugins

```python
app.plugins.has("git")          # True/False
app.plugins.get("git")          # the installed Plugin instance, or None
app.plugins.disable("git")      # stop it from receiving hooks (kept installed)
app.plugins.enable("git")       # re-enable it
app.plugins.remove("git")       # permanently uninstall it
app.plugins.list_plugins()      # [{"name": ..., "version": ..., "priority": ..., ...}, ...]
```

#### Error handling

By default, if one plugin's hook raises an exception, it's logged (via
`app.logger` if available) and every other plugin still runs — one
broken plugin won't take down the rest of your app.

```python
app.plugins.strict = True   # opt back into raise-immediately behavior
```

With `strict=True`, a hook error re-raises immediately as a `PluginError`
and aborts the rest of that dispatch — useful during development when you
want failures to be loud.

#### Built-in example plugins

`clidev/builtins/plugins.py` ships two reference plugins you can use as-is
or copy as a starting point:

```python
from clidev.builtins.plugins import GitPlugin, DatabasePlugin

app.use(GitPlugin())        # detects whether the cwd is a git repo
app.use(DatabasePlugin())   # confirms the storage backend is ready on start
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


## Clidev full example: "DevOps Toolkit"

Demonstrates nearly every feature covered in the README:

    - App setup with a theme
    - Banner splash screen
    - Menus (with nested submenu)
    - Forms with validation
    - Storage (persists project data to JSON)
    - Page routing
    - Conditional navigation (if_value / when)
    - Workflow engine (multi-step deploy pipeline)
    - Events (on_start / on_exit / on_submit)
    - Shell command execution + progress bar
    - Plugins (registry, config, dependencies)
    - Tables, cards, dashboard`
    - Image/video rendering (optional, only if deps installed)`

Run with:
    python examples/full_example.py

```python

from clidev import App
from clidev.plugins import Plugin, FunctionalPlugin

# ---------------------------------------------------------------------------
# App setup
# ---------------------------------------------------------------------------

app = App("DevOps Toolkit", theme="dark", storage_backend="json")


# ---------------------------------------------------------------------------
# Plugins
# ---------------------------------------------------------------------------

class GitPlugin(Plugin):
    name = "git"
    priority = 0

    def on_install(self, app):
        result = app.cmd("git rev-parse --is-inside-work-tree", capture=True)
        app.state["git_repo"] = result.ok

    def on_start(self, app):
        if app.state.get("git_repo"):
            app.logger.info("Git repository detected.")
        else:
            app.logger.warning("Not inside a git repository.")


class AnalyticsPlugin(Plugin):
    name = "analytics"
    priority = 50
    requires = ["git"]  # must be installed after GitPlugin

    def on_start(self, app):
        app.logger.info(f"Analytics session started (env: {self.get('env', 'dev')})")

    def on_submit(self, app, form_title, data):
        app.logger.info(f"[analytics] form submitted: {form_title}")

    def on_exit(self, app):
        app.logger.info("Analytics session ended.")


# Register once globally, reusable by name in any app
app.plugins.register_class("git", GitPlugin)
app.plugins.register_class("analytics", AnalyticsPlugin)

app.use("git")
app.use("analytics", env="production")

# Quick one-off plugin without a class
def greet(app):
    app.logger.info("Welcome to the DevOps Toolkit!")

app.use(FunctionalPlugin("greeter", on_start=greet))


# ---------------------------------------------------------------------------
# Events
# ---------------------------------------------------------------------------

@app.on_start
def startup():
    app.banner("DEVOPS", symbol="#").show()


@app.on_exit
def shutdown():
    app.info("Session ended. See you next time!")


@app.on_error
def handle_error(e):
    app.error(f"Something went wrong: {e}")


# ---------------------------------------------------------------------------
# Pages
# ---------------------------------------------------------------------------

@app.page("dashboard")
def dashboard_page():
    projects = app.storage.load("projects", default=[])

    dash = app.dashboard("Toolkit Overview")
    dash.panel("Projects", str(len(projects)))
    dash.panel("Git Repo", "Yes" if app.state.get("git_repo") else "No")
    dash.panel("Last Deploy", app.state.get("last_deploy", "never"))
    dash.show()

    if projects:
        table = app.table("Registered Projects", columns=["Name", "Language"])
        for p in projects:
            table.add_row(p["Project Name"], p["Language"])
        table.show()


@app.page("create_project")
def create_project_page():
    project_form = app.form("New Project")
    project_form.text("Project Name")
    project_form.select("Language", ["Python", "Rust", "Go"])
    project_form.checkbox("Use Docker")

    data = project_form.run()
    if not data:
        return  # user cancelled (Ctrl+C)

    app._last_form_data = data

    projects = app.storage.load("projects", default=[])
    projects.append(data)
    app.storage.save("projects", projects)

    app.success(f"Project '{data['Project Name']}' created.")

    # Conditional navigation based on form answers
    app.if_value("Language", equals="Python").goto("python_setup")
    app.if_value("Use Docker", equals=True).then(
        lambda: app.info("Docker setup will run on first deploy.")
    )


@app.page("python_setup")
def python_setup_page():
    with app.progress("Setting up Python environment"):
        app.cmd("python -m venv .venv_demo")
        app.cmd("echo done")
    app.success("Python environment ready.")


@app.page("deploy")
def deploy_page():
    pipeline = app.workflow("deploy_pipeline")

    def build(context):
        app.info("Building project...")
        result = app.cmd("echo Build complete", capture=True)
        return {"build_output": result.stdout.strip()}

    def test(context):
        app.info("Running tests...")
        return {"tests_passed": True}

    def deploy(context):
        if not context.get("tests_passed"):
            raise RuntimeError("Cannot deploy: tests failed")
        app.success(f"Deployed! ({context['build_output']})")
        return {"deployed": True}

    pipeline.step(build).step(test).step(deploy)

    with app.spinner("Running deploy pipeline..."):
        result = pipeline.start()

    if result.get("deployed"):
        import datetime
        app.state["last_deploy"] = datetime.datetime.now().strftime("%Y-%m-%d %H:%M")


@app.page("media_demo")
def media_demo_page():
    card = app.card(
        "Media Demo",
        "This page shows image/video rendering.\n"
        "Requires: pip install clidevkit[media]"
    )
    card.show()

    try:
        # Local file, URL, or Google Drive link all work here.
        app.image("assets/logo.png", size="medium").show()
    except Exception as e:
        app.warning(f"Skipping image demo: {e}")


# ---------------------------------------------------------------------------
# Menus (with a nested submenu)
# ---------------------------------------------------------------------------

main_menu = app.menu("DevOps Toolkit - Main Menu")
main_menu.option("Dashboard", "dashboard")
main_menu.option("Create Project", "create_project")
main_menu.option("Deploy", "deploy")
main_menu.option("Media Demo", "media_demo")

settings_menu = app.menu("Settings")
settings_menu.option("Show Plugins", lambda: app.console.print(app.plugins.list_plugins()))
settings_menu.option("Back", app.back)

main_menu.link("Settings", settings_menu)
main_menu.option("Exit", app.exit)


# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------

if __name__ == "__main__":
    app.run()
```python
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

All Rights Reserved
