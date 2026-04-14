# Encoders Plugin

A SimHub plugin that turns physical rotary encoders into smart in-car controls. The plugin reads HID buttons directly, syncs your encoder positions with in-game settings (ABS, TC, engine map and more), supports both incremental and absolute encoders, and can drive a Multi-Link mode where one encoder selects the role another encoder triggers. It also injects live overlays on top of any VoCore display so you can see the active mode at a glance from inside the cockpit.

## How It Works

1. Configure your encoder device in SimHub's Control Mapper with the increase/decrease roles you want to drive (for example `ABS+`/`ABS-`, `TractionControl+`/`-`, `FuelMix+`/`-`).
2. Add the encoder in the plugin and choose its type (ABS, TC, TC Cut, TC Slip, Engine Map).
3. The plugin reads HID button presses directly via [HidSharp](https://github.com/IntergatedCircuits/HidSharp), tracks a virtual position, and triggers the Control Mapper roles as needed so the in-game value matches what you have rotated to.
4. Sync runs only when the game is running and ignition is ON; it pauses automatically during replays, menu screens or when the car is not loaded.

## Features

### Encoder Modes
- **Incremental encoders** — separate CW/CCW buttons, one click per detent. Detect each direction with one click each, the plugin auto-discovers the Control Mapper roles from the device bindings.
- **Absolute encoders** — one HID button per physical position. Detect the first position once and the plugin computes the rest of the position-to-button mapping automatically.
- **Configurable detents** — 12, 16, 20 or any custom value defined in Settings.
- **Configurable start position** — 0 or 1 to match how each game numbers its values (ACC starts at 0, LMU at 1, etc.).
- **Calibrate** — for incremental encoders the calibration sets the current physical position as logical 0 (or 1). For absolute encoders the Calibrate button stores a virtual offset so any physical position can be remapped to logical 0 without re-running Detect — the badge shows the current offset in signed shortest form (for example `offset -1` for a 12-position encoder calibrated at physical 11).

### Encoder Types
The plugin reads the in-game value via SimHub telemetry for each supported type so the sync direction is always correct:

- ABS · TC · TC Cut · TC Slip · Engine Map

For TC Cut and TC Slip the plugin uses the most accurate per-game property path (iRacing, ACC, ACE, AC Rally, LMU, etc.).

### Game Sync Engine
- **Event-driven** — sync triggers on ignition ON, on car change, and on every encoder rotation. No periodic polling that hammers the game.
- **Shortest-path correction** — when the game value differs from the encoder, the plugin picks the shortest direction (increment vs decrement) to push the game to the right value.
- **Auto-pause** — sync is suspended during game pause, replay mode, or while the car is not loaded.
- **Live interrupt** — if you rotate the encoder while a sync sequence is mid-flight, it interrupts cleanly and restarts towards the new target.

### Multi-Link Mode
Multi-Link turns one encoder into a mode selector and another into the action knob. The master picks which role the slave triggers. No telemetry sync — purely a runtime mode router for users with one big rotary plus one small one.

- **Role Labels Template** — global table in Settings where you define reusable labels (Name + CW role + CCW role).
- **Master modes** — Left Only, Right Only, Separate (independent left/right masters), or Combined (one master controls both sides).
- **Per-position assignment** — click each dial position and pick a label, or leave it as `--` to make the slave inactive on that position.
- **Bind exclusivity validation** — the plugin warns you if you try to use the same physical button on more than one encoder per device.
- **SimHub properties exposed** — `MultiLinkLeft.CurrentMode`, `MultiLinkLeft.Triggered`, `MultiLinkLeft.AllLabels` (and the right-side equivalents) so dashboards and LED profiles can react to mode changes.

### VoCore Display Overlay
- **Per-screen configuration** — every screen of every dashboard on a VoCore can have its own label position, font size, orientation and colors. The plugin watches SimHub's `DashboardScreenStates` JSON files via `FileSystemWatcher` to know exactly which screen is active and applies the matching configuration in real time.
- **Friendly screen names** — the saved configs list shows real screen names (Race, Pit, Map, Timetables, etc.), looked up from the dashboard `.djson`. The full screen GUID is available as a tooltip.
- **Per-dashboard filtering** — the saved configs list only shows screens belonging to the dashboard currently rendering on the VoCore. Switching dashboards is detected within ~100 ms and the UI rebuilds instantly via an event-driven refresh.
- **Idle screens are filtered automatically** — overlays are hidden in idle anyway, so configuring them would be pointless.
- **Disconnected device card** — when a VoCore is offline, its card grays out and collapses; the toggle is disabled until the device is back online.
- **Configure Mode** — when no real game is running, a Configure Mode toggle appears at the top of the Overlay tab. Turning it ON injects a synthetic GameRunning state so the VoCore leaves idle and renders the in-game screens — you can position labels exactly where you want them without launching a real game. It auto-disables the moment a real game starts.
- **Direct DOM injection** — labels are drawn as styled `div`s inside the VoCore's CefSharp browser at z-index 999999990 (just below the WhatsApp plugin overlay).
- **Trigger flash** — labels flash a configurable color when the slave triggers a role, so you get visual confirmation of every action.

### SimHub Properties
The plugin exposes properties so dashboards, formulas and other plugins can react to its state:

- `EncodersPlugin.Encoder.{deviceId}.{index}.VirtualPosition` — current virtual position per encoder (with calibration / offset already applied)
- `EncodersPlugin.MultiLinkLeft.CurrentMode` / `.MultiLinkRight.CurrentMode` — active label name on each side
- `EncodersPlugin.MultiLinkLeft.Triggered` / `.MultiLinkRight.Triggered` — last triggered role name (auto-clears after ~1.5 s)
- `EncodersPlugin.MultiLinkLeft.AllLabels` / `.MultiLinkRight.AllLabels` — comma-separated list of all configured labels for that side

### Auto-Update
- **Automatic check on start** — every time you open the plugin in SimHub, it queries the GitHub Releases API and shows the result in the header next to the version number.
- **One-click in-place upgrade** — when a new version is available, the header lights up with `Version: X → Y  available  [Check Updates] [Download]`. Click Download to fetch the new DLL into `Documents\EncodersPlugin\updates\`, then Install & Restart to swap it. The plugin writes a small batch script that waits for SimHub to close, replaces the DLL in the SimHub folder, and reopens SimHub automatically.
- **Manual refresh** — click the Check Updates button at any time to re-query GitHub without restarting the plugin.
- **No external installer** — everything runs from inside the plugin; you never have to download or copy DLLs by hand after the first install.

### Other
- **Built-in Guide tab** — eight collapsible sections covering setup, both encoder modes, Multi-Link, the VoCore overlay, exposed SimHub properties, the auto-update flow, and troubleshooting.
- **Debug logging** — toggle from Settings; logs go to `Documents\EncodersPlugin\EncodersPlugin.log` and are cleared on every SimHub start.
- **Dark theme** — matches SimHub's native look with SimHub controls.

## Plugin Tabs

### Encoders
- Device dropdown (auto-discovered from Control Mapper) with online/offline indicator and Refresh button
- "+ Add Encoder" with per-encoder cards: label, type, detents, start position, mode, button bindings, calibration, live virtual position display
- Sub-tabs: **Encoders** (single-purpose encoders) and **Multi-Link** (mode-selector + action-knob setup)

### Overlay
- Configure Mode bar (visible only when no real game is running)
- One card per VoCore, with online/offline state, expand/collapse and per-screen configuration
- Per-screen list filtered to the current dashboard, with friendly screen names

### Settings
- Role Labels Template (global, reusable across all Multi-Link devices)
- Custom encoder detent counts
- Triggered flash duration for the VoCore overlay
- Debug logging toggle

### Guide
Eight collapsible sections that walk through the entire feature set: Getting Started, Adding & Configuring Encoders, Incremental Encoders, Absolute Encoders, Multi-Link Mode, VoCore Overlay, SimHub Properties, Troubleshooting.

### About
Version, links to releases, donation link, and links to the author's other SimHub plugins.

## Requirements

- [SimHub](https://www.simhubdash.com/)
- SimHub's **Control Mapper** plugin enabled, with your encoder device registered and at least one button mapped to a role
- Windows 10 / 11

## Installation

### First install

1. Download the latest `EncodersPlugin.dll` from [Releases](https://github.com/BrunoSilva1978PT/Encoders-SimHub-Plugin/releases)
2. Close SimHub
3. Copy the DLL to your SimHub installation folder (default: `C:\Program Files (x86)\SimHub\`)
4. Open SimHub
5. The plugin appears in the left menu as **Encoders Plugin**

### Updating

After the first install, you never need to download a DLL by hand again — the plugin updates itself.

1. Open the Encoders Plugin in SimHub
2. The header at the top right of the plugin window shows the current version. The plugin queries GitHub Releases automatically on every start; you can also click **Check Updates** to refresh manually.
3. When a new version is available, the header changes to `Version: X.Y.Z → A.B.C  available  [Check Updates] [Download]`
4. Click **Download** — the new DLL is fetched into `Documents\EncodersPlugin\updates\` with a live progress percentage
5. The button changes to **Install & Restart**. Click it.
6. The plugin writes a small batch script, closes SimHub, swaps the DLL in the SimHub folder, and reopens SimHub automatically with the new version

## Other Plugins by the Same Author

- [Remote Telemetry Plugin](https://github.com/BrunoSilva1978PT/Remote-Telemetry-Plugin) — share live SimHub dashboards between racing teammates over a secure WebSocket tunnel
- [WhatsApp SimHub Plugin](https://github.com/BrunoSilva1978PT/WhatsApp-SimHub-Plugin) — receive WhatsApp messages directly on your VoCore display from inside the cockpit

## License

MIT License
