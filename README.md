# HyprWindows for DankMaterialShell

A lightweight **Hyprland window switcher plugin for DankMaterialShell (DMS)** that lists open windows across all workspaces directly inside the DMS launcher.

It is designed to provide a workflow similar to:

```bash
rofi -show window
```

while staying completely inside the DankMaterialShell ecosystem.

The launcher displays windows in **MRU-style order**, prioritizes the previously focused window for quick switching, and works across Hyprland workspaces and layouts.

> [!NOTE]
> This project is **vibe coded**.  
> It was built iteratively with AI assistance, experimentation, debugging, and a healthy amount of "this should work... why doesn't this work?" energy.
>
> The code has been tested and refined for actual use, but contributions, fixes, and cleaner implementations are very welcome.

---

## Features

- Lists open Hyprland windows from **all workspaces**
- Uses the native **DankMaterialShell launcher**
- Search windows by:
  - Window title
  - Application class
  - Workspace
- Shows application icons using DMS desktop-entry lookup
- MRU-style window ordering
- Previously focused window appears first
- Current window appears second
- Remaining windows follow MRU order
- Exact window switching using Hyprland window addresses
- Works with multiple windows belonging to the same application
- Works across multiple monitors
- Supports floating windows
- Designed to work independently of the active Hyprland layout
- No dependency on Rofi, Wofi, Fuzzel, or another launcher

---

## Window Ordering

The plugin is intended to behave similarly to a traditional **Alt+Tab window switcher**.

Internally, the window history is maintained as:

```text
Current Window
Previous Window
Older Window
Older Window
...
```

The launcher presents that history as:

```text
Previous Window    ← automatically selected
Current Window
Next MRU Window
Next MRU Window
...
```

For example, suppose you switch between:

```text
Firefox     Workspace 1
VS Code     Workspace 4
Kitty       Workspace 2
Steam       Workspace 6
```

and Steam is currently focused.

Opening the window launcher will show:

```text
Kitty       Workspace 2    ← previous window
Steam       Workspace 6    ← current window
VS Code     Workspace 4
Firefox     Workspace 1
```

This makes quickly switching between the last two windows convenient.

---

## Multi-Workspace Support

HyprWindows maintains a **global MRU history based on Hyprland window addresses**.

The history is not separated by workspace.

This means the ordering remains consistent even when windows are distributed across many workspaces:

```text
Workspace 1 ─ Firefox

Workspace 2 ─ Kitty

Workspace 3 ─ Discord

Workspace 4 ─ VS Code

Workspace 5 ─ Steam
```

The plugin tracks the focus order globally rather than grouping windows by workspace.

---

## Hyprland Layout Support

Window switching uses the exact Hyprland window address rather than directional layout navigation.

The plugin therefore does not depend on commands such as:

```text
focus left
focus right
focus next
```

which can behave differently depending on the active layout.

The plugin is intended to work with windows using layouts such as:

- Dwindle
- Master
- Scrolling
- Grouped / tabbed windows
- Floating windows

as well as windows located on different workspaces or monitors.

---

## How Window Switching Works

Each Hyprland client has a unique address, for example:

```text
0x55c3ab72c010
```

HyprWindows stores this address with the launcher item.

Selecting the window then focuses that exact Hyprland client.

On current Hyprland versions the plugin uses the equivalent of:

```lua
hl.dsp.focus({
    window = "address:0x55c3ab72c010"
})
```

Using the address avoids ambiguity when multiple windows have:

- The same application class
- Similar titles
- Multiple instances of the same application

---

## Requirements

You need:

- [Hyprland](https://hypr.land/)
- [DankMaterialShell](https://github.com/AvengeMedia/DankMaterialShell)
- `hyprctl`

Recommended:

```text
DMS >= 1.4.0
```

The plugin uses DMS launcher functionality that allows plugin-provided result ordering.

---

## Installation

Clone the repository:

```bash
git clone https://github.com/shans10/HyprWindows.git
```

Copy the plugin into the DMS plugins directory:

```bash
mkdir -p ~/.config/DankMaterialShell/plugins

cp -r HyprWindows \
    ~/.config/DankMaterialShell/plugins/HyprWindows
```

Your directory should look like:

```text
~/.config/DankMaterialShell/plugins/HyprWindows/
├── plugin.json
├── HyprWindowsLauncher.qml
├── StartupCheck.qml
└── README.md
```

Then scan for plugins:

```bash
dms ipc plugin-scan scan
```

Reload HyprWindows:

```bash
dms ipc plugin-scan reload hyprWindows
```

If necessary, restart DMS:

```bash
dms restart
```

Then enable **Hyprland Windows** from the DMS plugin settings.

---

## Usage

Open the DankMaterialShell launcher and type:

```text
!
```

All currently available Hyprland windows will appear.

### Search by application

```text
! firefox
```

### Search by window title

```text
! youtube
```

### Search by application class

```text
! code
```

### Search by workspace

```text
! 3
```

Select a result and press **Enter** to switch to that window.

---

## Recommended Keybind

You can make HyprWindows behave like a dedicated window switcher by opening DMS directly with the `!` query.

### Hyprland Lua configuration

For example:

```lua
hl.bind(
    "SUPER + TAB",
    hl.dsp.exec_cmd([[dms ipc call spotlight openQuery "!"]])
)
```

or:

```lua
hl.bind(
    "ALT + TAB",
    hl.dsp.exec_cmd([[dms ipc call spotlight openQuery "!"]])
)
```

Pressing the shortcut immediately opens DMS in window-switching mode.

---

## Search

The plugin searches against a combination of:

```text
Application Class
Window Title
Workspace
```

A lightweight fuzzy/subsequence matcher is also used.

For example:

```text
!fire
```

may match:

```text
Firefox - GitHub
Firefox - YouTube
Firefox - ChatGPT
```

The filtered results retain their MRU ordering.

---

## MRU Tracking

HyprWindows maintains its own in-memory MRU list using **window addresses**.

When focus changes:

```text
A → B → C
```

the internal history becomes:

```text
C
B
A
```

When the launcher opens while `C` is focused, it presents:

```text
B    ← previous
C    ← current
A
```

When `B` is selected:

```text
B
C
A
```

becomes the new internal MRU order.

Opening the launcher again therefore produces:

```text
C    ← previous
B    ← current
A
```

---

## Initial MRU State

The plugin's custom MRU history exists in memory.

Immediately after:

```bash
dms restart
```

the plugin does not yet have its own previous focus history.

During startup it therefore uses Hyprland's existing focus information to initialize the MRU list.

Once normal window switching begins, HyprWindows maintains its own ordering.

---

## Why Not Just Use `focusHistoryID`?

Hyprland exposes focus history information through its client data.

That works well for obtaining an initial window ordering, but relying exclusively on compositor-provided history can produce behavior that differs depending on workspace state and how windows are organized.

HyprWindows therefore uses Hyprland's history primarily to initialize its state, then maintains a simple address-based global MRU list itself.

This also makes the window-switching logic easier to reason about:

```text
focus window
      ↓
move its address to MRU position 0
      ↓
old current window becomes position 1
      ↓
launcher displays position 1 first
```

---

## Why DankMaterialShell?

There are already excellent window switchers available for Wayland and Hyprland:

- Rofi
- Fuzzel
- Wofi
- Walker
- Various Hyprland-specific scripts

The goal of this plugin isn't to replace them.

The goal is simply:

> **If DMS is already my shell and launcher, why should I need another launcher just to switch windows?**

HyprWindows keeps application launching and window switching inside the same UI.

---

## Project Structure

```text
HyprWindows/
├── HyprWindowsLauncher.qml
├── StartupCheck.qml
├── plugin.json
└── README.md
```

### `HyprWindowsLauncher.qml`

Contains the main plugin logic:

- Fetching Hyprland clients
- MRU tracking
- Searching
- Launcher result generation
- Sorting
- Window focusing

### `StartupCheck.qml`

Checks that:

```text
hyprctl
```

is available and that a Hyprland session can be queried.

### `plugin.json`

Defines the DMS plugin metadata, launcher trigger, dependencies, and required permissions.

---

## Known Limitations

### MRU history resets with DMS

The custom MRU history currently lives in memory.

Running:

```bash
dms restart
```

will rebuild the initial history from Hyprland client information.

### Polling

The current implementation periodically queries:

```bash
hyprctl -j clients
```

to keep the window cache and active-window state updated.

A future implementation could instead listen directly to Hyprland's event socket.

---

## Possible Future Improvements

Some things that may be worth adding:

- Hyprland socket2 event support
- Event-driven MRU tracking instead of polling
- Persistent MRU history
- Configurable launcher trigger
- Configurable sorting behavior
- Option to hide the currently focused window
- Workspace icons
- Monitor information
- Special workspace support
- Better application icon matching
- Window thumbnails
- Configurable display format
- More advanced fuzzy matching
- Close-window action
- Move-window-to-workspace actions

---

## Troubleshooting

### Plugin does not appear

Run:

```bash
dms ipc plugin-scan scan
```

then:

```bash
dms ipc plugin-scan reload hyprWindows
```

If that does not help:

```bash
dms restart
```

---

### Windows appear but selecting one does nothing

Check that Hyprland itself can query clients:

```bash
hyprctl -j clients
```

Also verify that you are running inside the intended Hyprland session.

---

### Old plugin behavior remains after updating

DMS may still have the previous QML instance loaded.

Restart DMS:

```bash
dms restart
```

---

## Contributing

Issues and pull requests are welcome.

In particular, testing on different setups would be useful:

- Multiple monitors
- Many workspaces
- Scrolling layout
- Dwindle layout
- Master layout
- Grouped/tabbed windows
- Special workspaces
- Mixed tiled and floating windows

If something behaves strangely, please include:

```bash
hyprctl version
dms --version
```

along with a description of your workspace/window layout.

---

## Vibe Coding Disclaimer

Yes, this project is **vibe coded**.

The original plugin was created through an iterative conversation with an AI coding assistant:

```text
idea
 ↓
generate something
 ↓
run it
 ↓
discover weird edge case
 ↓
"okay that doesn't work"
 ↓
change architecture
 ↓
discover an even weirder Hyprland edge case
 ↓
repeat
```

The plugin exists because I wanted DMS to have a simple all-workspace window switcher similar to Rofi's window mode and decided to build one.

The goal is not to pretend every line was meticulously handcrafted in isolation. The goal is to make something useful, understand the problems encountered along the way, and improve it when those problems show up.

If you find questionable code, there is therefore a perfectly reasonable explanation:

**the vibes were strong that day.**

PRs improving said vibes are welcome.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## Acknowledgements

Built for:

- [DankMaterialShell](https://github.com/AvengeMedia/DankMaterialShell)
- [Hyprland](https://hypr.land/)

Inspired by the convenience of:

```bash
rofi -show window
```

and the desire to have the same workflow directly inside DMS.