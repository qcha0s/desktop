# SDL: Add `on_close` Option to `Desktop.Window`

**Author:** Robert French
**Date:** 2026-02-23
**Status:** Proposed
**Affects:** `Desktop.Window`

---

## Problem

Applications that use multiple `Desktop.Window` instances have no way to control
what happens when a user clicks the native close button (the red X on macOS, the
X on Windows/Linux) on a secondary window.

The current `handle_cast(:close_window, ...)` implementation decides the close
behavior based on the presence of a `taskbar` (which is derived from the
`icon_menu` option):

```elixir
if taskbar == nil do
  OS.shutdown()    # terminates the entire application
else
  :wxFrame.hide(frame)
end
```

This means the **only** way to get hide-on-close behavior is to provide an
`icon_menu`, which creates a system tray icon. For secondary windows like
Settings or Preferences panels, this is inappropriate — they should be
dismissible without either terminating the application or requiring a taskbar
icon.

### Impact

On macOS, `OS.shutdown()` during wxWidgets teardown of a secondary window causes
a segmentation fault (SIGSEGV, exit code 139). On other platforms it causes an
unexpected full application exit when the user only intended to close a secondary
window.

---

## Solution

Add an explicit `on_close` configuration option to `Desktop.Window` that
decouples the close behavior from the taskbar presence.

### New Option

| Option | Values | Default | Description |
|---|---|---|---|
| `:on_close` | `:quit`, `:hide` | `:quit` | Controls behavior when the native close button is clicked |

- **`:quit`** — Existing behavior. Shuts down the application (via
  `OS.shutdown()`). Appropriate for primary/main windows.
- **`:hide`** — Hides the window frame. Appropriate for secondary windows
  (settings, preferences, tool panels) that should be dismissible without
  terminating the application.

### Usage

```elixir
children = [
  # Primary window — closes the app (default behavior, unchanged)
  {Desktop.Window,
   [
     app: :my_app,
     id: MainWindow,
     title: "My App",
     icon_menu: MyApp.IconMenu,
     url: &MyAppWeb.Endpoint.url/0
   ]},

  # Secondary window — hides on close
  {Desktop.Window,
   [
     app: :my_app,
     id: SettingsWindow,
     title: "Settings",
     hidden: true,
     on_close: :hide,
     url: fn -> MyAppWeb.Endpoint.url() <> "/settings" end
   ]}
]
```

---

## Changes

### 1. `%Desktop.Window{}` struct (`lib/desktop/window.ex`)

Add `on_close` field with a default of `:quit` for full backwards compatibility:

```elixir
defstruct [
  :module,
  :taskbar,
  :frame,
  :notifications,
  :webview,
  :home_url,
  :last_url,
  :title,
  on_close: :quit
]
```

### 2. `init/1` (`lib/desktop/window.ex`)

Read the option from the configuration keywords and store it in the struct:

```elixir
on_close = options[:on_close] || :quit

ui = %Window{
  frame: frame,
  webview: Fallback.webview_new(frame),
  notifications: %{},
  home_url: url,
  title: window_title,
  taskbar: taskbar,
  on_close: on_close
}
```

### 3. `handle_cast(:close_window, ...)` (`lib/desktop/window.ex`)

Check `on_close` before falling through to the existing logic:

```elixir
def handle_cast(:close_window, ui = %Window{frame: frame, taskbar: taskbar, on_close: on_close}) do
  if on_close == :hide do
    :wxFrame.hide(frame)
    {:noreply, ui}
  else
    # Existing behavior preserved exactly as-is
    if not :wxFrame.isShown(frame) do
      OS.shutdown()
    end

    if taskbar == nil do
      OS.shutdown()
      {:noreply, ui}
    else
      :wxFrame.hide(frame)
      {:noreply, ui}
    end
  end
end
```

### 4. `@moduledoc` (`lib/desktop/window.ex`)

Document the new option alongside the existing configuration options:

```
* `:on_close` - controls the behavior when the native window close
  button is clicked.

    Possible values are:

    * `:quit` - Shut down the application (default). This is the
      legacy behavior and is appropriate for main/primary windows.

    * `:hide` - Hide the window instead of quitting. Useful for
      secondary windows (e.g. settings, preferences) that should
      be dismissible without terminating the application.
```

### 5. `desktop.install` mix task (`lib/mix/tasks/desktop.install.ex`)

Guard the module definition with `Code.ensure_loaded?/1` so the optional
`igniter` dependency doesn't cause compilation failures when Desktop is used as
a path dependency:

```elixir
if Code.ensure_loaded?(Igniter.Mix.Task) do
  defmodule Mix.Tasks.Desktop.Install do
    # ...
  end
end
```

---

## Backwards Compatibility

- The default value of `on_close` is `:quit`, which preserves the exact
  existing behavior for all current users.
- No existing configuration options are modified or removed.
- The `taskbar`-based logic is unchanged when `on_close` is `:quit`.
- Applications that do not pass `on_close` will behave identically to
  previous versions.

---

## Testing

Manual verification:

1. **Primary window with `on_close: :quit` (default):** Clicking the close
   button shuts down the application — unchanged behavior.
2. **Secondary window with `on_close: :hide`:** Clicking the close button
   hides the window. The application continues running. The window can be
   re-shown via `Desktop.Window.show/2`.
3. **macOS Cmd+Q:** Application quits normally regardless of `on_close`
   setting on any window.
