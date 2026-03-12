# SDL: macOS App Menu Support and Window Sizing Options

**Author:** Robert French
**Date:** 2026-02-23
**Status:** Proposed
**Affects:** `Desktop.Window`, `Desktop.Wx`, `desktop_wx.erl`, `Desktop.Menu.Adapter.Wx`

---

## Problem

### 1. macOS App Menu Items

On macOS, standard application conventions place About, Settings/Preferences,
and Quit in the **application menu** (the menu bearing the app's name, to the
right of the Apple menu). The File menu should contain only document-related
actions like Close Window.

Desktop's menu system creates all items with `wxID_ANY()`, which means
wxWidgets has no way to identify special items and auto-place them in the
macOS app menu. All items stay wherever the developer puts them in the XML,
resulting in a non-standard menu layout.

Additionally, `Desktop.Window` did not retain a reference to the Menu
GenServer started during init. This meant that even if items were placed in
the macOS app menu (via wxWidgets special IDs), their click events would be
caught by the frame-level event handler in `Desktop.Window.handle_event/2`,
which only handled `wxID_EXIT` and silently dropped all other events.

### 2. Fixed-Size Windows

There is no way to prevent a `Desktop.Window` from being resized. The init
function supports `min_size` but not `max_size`, so windows like About panels
that should be fixed-size cannot be locked.

---

## Solution

### 1. Menu Item Roles

Add a `role` attribute to the `<item>` tag in the menu XML that maps to
wxWidgets special menu item IDs. On macOS, items with these IDs are
automatically placed in the application menu by wxWidgets.

| Role | wxWidgets ID | macOS Behavior |
|---|---|---|
| `"about"` | `wxID_ABOUT` | Placed in app menu as "About [App Name]" |
| `"preferences"` | `wxID_PREFERENCES` | Placed in app menu as "Settings..." |
| `"quit"` | `wxID_EXIT` | Placed in app menu as "Quit [App Name]" |
| (none) | `wxID_ANY` | Stays in its declared menu (default, unchanged) |

### 2. Menu Event Forwarding

Store the Menu GenServer pid in the `Desktop.Window` struct and forward
macOS app menu events (`wxID_ABOUT`, `wxID_PREFERENCES`) to the Menu module
via `Desktop.Menu.trigger_event/2`.

### 3. Max Size

Add `max_size` option to `Desktop.Window`, mirroring the existing `min_size`.
Setting both to the same value as `size` creates a fixed-size window.

---

## Changes

### 1. `Desktop.Wx` (`lib/desktop/wx.ex`) + `desktop_wx.erl` (`src/desktop_wx.erl`)

Add `ID_ABOUT` and `ID_PREFERENCES` to the constants list so the
corresponding `wxID_ABOUT()` and `wxID_PREFERENCES()` functions are generated.
The Erlang source `desktop_wx.erl` also gains the matching `get/1` clauses:

```elixir
@constants ~w(
  ID_ANY ID_EXIT ID_ABOUT ID_PREFERENCES DEFAULT_FRAME_STYLE ...
)
```

```erlang
get(wxID_ABOUT) -> ?wxID_ABOUT;
get(wxID_PREFERENCES) -> ?wxID_PREFERENCES;
```

### 2. `Desktop.Menu.Adapter.Wx` (`lib/desktop/menu/adapters/wx.ex`)

Map the `role` attribute to the correct wxWidgets ID when creating menu items:

```elixir
{:item, attr, content} ->
  # ...
  id = role_to_id(attr[:role])
  item = :wxMenuItem.new(id: id, text: List.flatten(content), kind: kind)
```

With the helper:

```elixir
defp role_to_id("about"), do: Wx.wxID_ABOUT()
defp role_to_id("preferences"), do: Wx.wxID_PREFERENCES()
defp role_to_id("quit"), do: Wx.wxID_EXIT()
defp role_to_id(_), do: Wx.wxID_ANY()
```

### 3. `%Desktop.Window{}` struct (`lib/desktop/window.ex`)

Add `menu_pid` field to retain a reference to the Menu GenServer:

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
  :menu_pid,
  on_close: :quit
]
```

### 4. `init/1` (`lib/desktop/window.ex`)

Store the Menu GenServer pid and read `max_size`:

```elixir
max_size = options[:max_size]

{wx_menubar, menubar_pid} =
  if menubar do
    {:ok, menu_pid} = Menu.start_link(...)
    wx_menubar = Menu.menubar(menu_pid)
    :wxFrame.setMenuBar(frame, wx_menubar)
    {wx_menubar, menu_pid}
  else
    {nil, nil}
  end

# ...

if max_size do
  :wxFrame.setMaxSize(frame, max_size)
end

ui = %Window{
  # ...
  menu_pid: menubar_pid,
  # ...
}
```

### 5. `handle_event/2` for `command_menu_selected` (`lib/desktop/window.ex`)

Forward macOS app menu events to the Menu GenServer:

```elixir
def handle_event(
      wx(id: id, event: wxCommand(type: :command_menu_selected)),
      ui = %Window{menu_pid: menu_pid}
    ) do
  cond do
    id == Wx.wxID_EXIT() ->
      quit()

    id == Wx.wxID_ABOUT() and menu_pid != nil ->
      Menu.trigger_event(menu_pid, "about")

    id == Wx.wxID_PREFERENCES() and menu_pid != nil ->
      Menu.trigger_event(menu_pid, "preferences")

    true ->
      :ok
  end

  {:noreply, ui}
end
```

### 6. `update_apple_menu/3` guard (`lib/desktop/window.ex`)

`update_apple_menu` is now only called for windows that have a menubar
(`wx_menubar != nil`). Previously it ran for every window on macOS, creating
a throwaway `wxMenuBar.new()` for windows without one (e.g., an About panel),
which wasted resources and overwrote the Apple menu title with the wrong
window's name.

### 7. `update_apple_menu/3` implementation (`lib/desktop/window.ex`)

Preserve About and Preferences items in the macOS app menu instead of
deleting them:

```elixir
defp update_apple_menu(title, frame, menubar) do
  menu = :wxMenuBar.oSXGetAppleMenu(menubar)
  :wxMenu.setTitle(menu, title)

  keep_ids = [Wx.wxID_EXIT(), Wx.wxID_ABOUT(), Wx.wxID_PREFERENCES()]

  for item <- :wxMenu.getMenuItems(menu) do
    id = :wxMenuItem.getId(item)

    cond do
      id == Wx.wxID_EXIT() ->
        :wxMenuItem.setText(item, "Quit #{title}\tCtrl+Q")

      id in keep_ids ->
        :ok

      true ->
        :wxMenu.delete(menu, item)
    end
  end

  :wxFrame.connect(frame, :command_menu_selected)
end
```

---

## Usage

### Menu XML with Roles

```elixir
def render(_assigns) do
  {:safe,
   """
   <menubar>
     <menu label="My App">
       <item role="about" onclick="about">About My App</item>
       <item role="preferences" onclick="settings">Settings\tCtrl+,</item>
       <item role="quit" onclick="quit">Quit My App\tCtrl+Q</item>
     </menu>
     <menu label="File">
       <item onclick="close">Close Window\tCtrl+W</item>
     </menu>
   </menubar>
   """}
end
```

On macOS, wxWidgets will auto-move the role-tagged items to the app menu.
On other platforms, the items remain where declared.

### Fixed-Size Window

```elixir
{Desktop.Window,
 [
   app: :my_app,
   id: AboutWindow,
   title: "About",
   size: {440, 460},
   min_size: {440, 460},
   max_size: {440, 460},
   hidden: true,
   on_close: :hide,
   url: fn -> MyAppWeb.Endpoint.url() <> "/about" end
 ]}
```

---

## Backwards Compatibility

- Items without a `role` attribute continue to use `wxID_ANY()` — no
  behavior change.
- The `menu_pid` field defaults to `nil`. Windows without a menubar are
  unaffected.
- `max_size` is optional and defaults to `nil` (no maximum), preserving
  existing resize behavior.
- The `update_apple_menu` function's existing behavior for `wxID_EXIT`
  is unchanged.
- The `handle_event` for `command_menu_selected` still calls `quit()` for
  `wxID_EXIT` — unchanged behavior.
- Event forwarding only occurs when `menu_pid` is not nil, so windows
  without menus are unaffected.

---

## Testing

Manual verification:

1. **About menu item (macOS app menu):** Clicking "About [App]" in the app
   menu opens the About window.
2. **Settings menu item (macOS app menu):** Clicking "Settings" or pressing
   Cmd+, opens the Settings window.
3. **Quit (macOS app menu):** Clicking "Quit [App]" or pressing Cmd+Q
   terminates the application.
4. **File > Close Window:** Clicking "Close Window" or pressing Cmd+W
   hides the main window.
5. **Fixed-size window:** About window cannot be resized by dragging edges
   or corners.
6. **Non-macOS platforms:** All menu items remain in their declared menus
   and function correctly via the standard adapter event path.
