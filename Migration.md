# Ryoku Arch → Niri Migration Plan

## Overview

This document provides a comprehensive migration plan for replacing Hyprland with Niri as the compositor in Ryoku Arch. The migration is a **clean break** - no dual support, no rollback to Hyprland.

**Status:** Planning Phase (awaiting implementation)
**Target:** Niri scrollable-tiling Wayland compositor
**Timeline:** 40 days (part-time)
**Testing:** QEMU VMs with vm-curator

---

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Migration Strategy | Clean break | Single codebase, simpler maintenance |
| Hyprland Plugin Packages | Delete entirely | Original repo has them, no need to maintain |
| Hub Backend (`hypr.go`) | Keep as reference | Historical reference, not used |
| Keybinds | Default Niri + extras | Focus on Ryoku-specific features |
| Animations | Defer to Phase 2 | Ship stable first, polish later |
| Workspace Keybinds | Niri defaults (Mod+I/U) | Niri's model is different (scrollable) |
| Autostart | systemd user services | Better dependency management |
| `hyprpicker` | Test first | May work with Niri, defer replacement |
| `hyprland-preview-share-picker` | Keep | It's compositor-agnostic |

---

## Architecture

### Current State (Hyprland)

```
┌─────────────────────────────────────┐
│         Ryoku Desktop Shell         │ (Quickshell QML - UNCHANGED)
│         Ryoku Hub (UI)              │ (Quickshell QML - UNCHANGED)
└─────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────┐
│         Ryoku Hub (Backend)         │ (Go: hypr.go)
│         hyprctl IPC                 │
└─────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────┐
│         Hyprland (Lua Config)       │ (ryoku/hyprland/)
│         hyprland.lua + modules/     │
└─────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────┐
│         Hyprland Compositor         │ (REPLACE)
│         xdg-desktop-portal-hypr     │ (REPLACE)
└─────────────────────────────────────┘
```

### Target State (Niri)

```
┌─────────────────────────────────────┐
│         Ryoku Desktop Shell         │ (Quickshell QML - UNCHANGED)
│         Ryoku Hub (UI)              │ (Quickshell QML - UNCHANGED)
└─────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────┐
│         Ryoku Hub (Backend)         │ (Go: niri.go)
│         niri msg IPC                │
└─────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────┐
│         Niri Config (KDL)           │ (ryoku/niri/)
│         config.kdl + user.kdl       │
└─────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────┐
│         Niri Compositor             │ (NEW)
│         xdg-desktop-portal-gnome    │ (NEW)
└─────────────────────────────────────┘
```

---

## Phase 1: Package Layer (Days 1-3)

### 1.1 Update `system/packages/base.packages`

**Remove:**
```
hyprland
xdg-desktop-portal-hyprland
hyprpolkitagent
hypridle
hyprpicker
hyprsunset
```

**Add:**
```
# Compositor
niri
xwayland-satellite

# Portals
xdg-desktop-portal-gnome
xdg-desktop-portal-gtk

# Alternatives
swayidle
wlsunset
grim
slurp
```

### 1.2 Update `release/packages/ryoku-desktop/PKGBUILD`

**Remove from `depends=`:**
```bash
'hyprland'
'xdg-desktop-portal-hyprland'
'hyprpolkitagent'
'hypridle'
'hyprpicker'
'hyprsunset'
"hypr-dynamic-cursors=$pkgver"
"ryoku-hypr-plugins=$pkgver"
"hyprglass=$pkgver"
"imgborders=$pkgver"
```

**Add to `depends=`:**
```bash
'niri'
'xwayland-satellite'
'xdg-desktop-portal-gnome'
'xdg-desktop-portal-gtk'
'swayidle'
'wlsunset'
'grim'
'slurp'
```

### 1.3 Delete Obsolete Packages

**Remove directories:**
```bash
rm -rf release/packages/hypr-dynamic-cursors/
rm -rf release/packages/ryoku-hypr-plugins/
rm -rf release/packages/hyprglass/
rm -rf release/packages/imgborders/
```

**Decision point:** Also delete `release/packages/hyprland-preview-share-picker/`? (Recommended: Keep, it's compositor-agnostic)

### 1.4 Validation

- [ ] `pacman -Syu` resolves dependencies correctly
- [ ] No circular dependencies
- [ ] Package builds successfully

---

## Phase 2: Niri Configuration (Days 4-10)

### 2.1 Create Directory Structure

```
ryoku/niri/
├── config.kdl              # Main config (seeded)
├── user.kdl.example        # User override template
└── scripts/
    └── (future helpers)
```

### 2.2 Config Porting Strategy

| Hyprland Module | Niri Equivalent | Port? | Notes |
|-----------------|-----------------|-------|-------|
| `env.lua` | `spawn-at-startup` | Yes | Environment, services |
| `keyboard.lua` | `input { keyboard {} }` | Yes | Layout, options |
| `monitors.lua` | `output "NAME" {}` | Yes | Critical |
| `displays.lua` | `output "NAME" {}` | Yes | Merge with monitors |
| `input.lua` | `input { touchpad, mouse }` | Yes | Touchpad, mouse |
| `decoration.lua` | `layout {}` | Yes | Gaps, borders, shadows |
| `binds.lua` | `binds {}` | Partial | Extras only (~30 keybinds) |
| `window_rules.lua` | `window-rule {}` | Yes | Float, workspace rules |
| `fullscreen.lua` | Built-in | No | Niri has Mod+Shift+F |
| `autostart.lua` | systemd services | Yes | Complex dependencies |
| `lid.lua` | N/A | No | Niri handles differently |
| `record.lua` | Defer | No | Phase 2 |
| `ryoshot.lua` | N/A | No | Quickshell, unchanged |
| `perf_saver.lua` | N/A | No | Different perf model |

### 2.3 Keybind Porting (Partial)

**Port from `binds.lua` - extras only, skip standard window management:**

```kdl
// Shell surfaces
"Super+Space" = { spawn = "qs -c shell" }
"Super+L" = { spawn = "ryoku-shell lock" }
"Super+Escape" = { spawn = "qs -c quicksettings" }
"Super+W" = { spawn = "qs -c wallpaper-menu" }
"Super+Shift+W" = { spawn = "ryoku-shell wallpaper random" }
"Super+V" = { spawn = "qs -c clipboard" }
"Super+Tab" = { spawn = "qs -c overview" }
"Super+M" = { spawn = "qs -c visualizer" }
"Super+Shift+M" = { spawn = "qs -c visualizer-overlay" }
"Super+Alt+M" = { spawn = "qs -c visualizer-place" }
"Super+grave" = { spawn = "ryoku-shell voice" }
"Super+comma" = { spawn = "qs -c hub open" }
"Super+S" = { spawn = "qs -c stash" }
"Super+Shift+S" = { spawn = "flock -n -o /tmp/ryoshot.lock qs -c ryoshot" }
"Super+Shift+C" = { spawn = "hyprpicker -a" } // Test if works, else grimblast

// Apps
"Super+Return" = { spawn = "ryoku-app terminal" }
"Super+E" = { spawn = "ryoku-app files" }
"Super+B" = { spawn = "ryoku-app browser" }
"Super+N" = { spawn = "ryoku-app editor" }
"Super+O" = { spawn = "ryoku-app notes" }
"Super+Alt+E" = { spawn = "kitty -e yazi" }

// Media keys
"XF86AudioRaiseVolume" = { spawn = "ryoku-volume up" }
"XF86AudioLowerVolume" = { spawn = "ryoku-volume down" }
"XF86AudioMute" = { spawn = "wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle" }
"XF86AudioPlay" = { spawn = "playerctl play-pause" }
"XF86AudioNext" = { spawn = "playerctl next" }
"XF86AudioPrev" = { spawn = "playerctl previous" }

// Brightness
"XF86MonBrightnessUp" = { spawn = "ryoku-cmd-brightness +5" }
"XF86MonBrightnessDown" = { spawn = "ryoku-cmd-brightness -5" }

// Touchpad
"XF86TouchpadToggle" = { spawn = "ryoku-cmd-touchpad toggle" }
"XF86TouchpadOn" = { spawn = "ryoku-cmd-touchpad on" }
"XF86TouchpadOff" = { spawn = "ryoku-cmd-touchpad off" }
```

**Decision:** Use Niri's default keybinds for workspace navigation (Mod+I/U) instead of Hyprland's Super+1-0 scheme.

### 2.4 Window Rules Porting

From `window_rules.lua` to Niri `window-rule {}`:

```kdl
window-rule {
    class = "kitty"
    float = true
}

window-rule {
    class = "firefox"
    workspace = "2"  // Use indices for now
}

// Layer rules
layer-rule {
    class = "some-app"
    layer = "top"
}
```

**Decision point:** Use workspace names or indices? (Recommended: Indices for now)

### 2.5 Autostart Porting

**Option A: Niri `spawn-at-startup`** (simpler)
**Option B: systemd user services** (recommended for complex deps)

**Example systemd approach:**

Create `ryoku/shell/systemd/user/niri-session.target`:
```ini
[Unit]
Description=Niri Session Target
After=niri.service xdg-desktop-portal.service

[Target]
Wants=ryoku-shell.service xdg-desktop-portal-gnome.service
```

Update `ryoku-shell.service`:
```ini
[Unit]
Description=Ryoku Shell
After=niri-session.target

[Service]
Type=exec
ExecStart=/usr/bin/qs -c shell
Restart=on-failure
```

### 2.6 Validation

- [ ] `niri validate` passes
- [ ] All keybinds work
- [ ] Window rules apply correctly
- [ ] Autostart services launch

---

## Phase 3: Hub Backend (Days 11-17)

### 3.1 Create `ryoku/hub/backend/niri.go`

**Based on `hypr.go` as reference, adapt for Niri:**

```go
package backend

import (
    "encoding/json"
    "os/exec"
    "os"
    "path/filepath"
    "strings"
    "io/ioutil"
)

// GenerateConfig generates KDL config from Hub settings
func GenerateConfig(options *HubOptions) (string, error) {
    // Build KDL string
    var kdl strings.Builder
    
    kdl.WriteString("input {\n")
    // ... write keyboard, touchpad, mouse
    
    kdl.WriteString("}\n")
    // ... write outputs, layout, binds, etc.
    
    return kdl.String(), nil
}

// ReloadConfig reloads Niri config
func ReloadConfig() error {
    cmd := exec.Command("niri", "msg", "config-reload")
    return cmd.Run()
}

// GetActiveWindow returns focused window info
func GetActiveWindow() (Window, error) {
    cmd := exec.Command("niri", "msg", "--json", "focused-window")
    output, err := cmd.Output()
    if err != nil {
        return Window{}, err
    }
    
    var result map[string]interface{}
    if err := json.Unmarshal(output, &result); err != nil {
        return Window{}, err
    }
    
    // Parse and return Window struct
    // ...
}

// SetCursor sets cursor theme (writes to config, then reloads)
func SetCursor(theme string, size int) error {
    configPath := filepath.Join(os.Getenv("HOME"), ".config", "niri", "config.kdl")
    content, err := ioutil.ReadFile(configPath)
    if err != nil {
        return err
    }
    
    // Update cursor section in KDL
    // ...
    
    if err := ioutil.WriteFile(configPath, newContent, 0644); err != nil {
        return err
    }
    
    return ReloadConfig()
}
```

### 3.2 Hub Main Integration

**File:** `ryoku/hub/backend/main.go`

**Change:**
```go
import "ryoku/hub/backend/niri"

func main() {
    // Use niri backend
    // hypr.go kept as reference but not imported
}
```

**Add comment in `main.go`:**
```go
// Note: hypr.go is kept as a reference for the old Hyprland implementation.
// It is no longer used in the codebase but may be useful for historical context.
```

### 3.3 Hub UI Pages (unchanged, backend adaptation only)

- KeybindsPage.qml → Backend generates KDL keybinds
- DisplaysPage.qml → Backend writes `output {}` blocks
- InputPage.qml → Backend writes `input {}` blocks
- AppearancePage.qml → Backend writes `layout {}`, `animations {}`
- WindowRulesPage.qml → Backend writes `window-rule {}` blocks
- AutostartPage.qml → Backend writes systemd services

### 3.4 Validation

- [ ] Hub UI opens and all pages load
- [ ] Config changes apply live
- [ ] `niri msg config-reload` works
- [ ] IPC commands return correct data

---

## Phase 4: Session & Autostart (Days 18-21)

### 4.1 Update Session Startup

**Create:** `ryoku/shell/systemd/user/niri-session.target`

```ini
[Unit]
Description=Niri Session Target
After=niri.service xdg-desktop-portal.service

[Target]
Wants=ryoku-shell.service xdg-desktop-portal-gnome.service
```

**Update:** `ryoku/shell/systemd/user/ryoku-shell.service`

```ini
[Unit]
Description=Ryoku Shell
After=niri-session.target

[Service]
Type=exec
ExecStart=/usr/bin/qs -c shell
Restart=on-failure
```

**Update:** `xdg-desktop-portal.service`

```ini
[Unit]
Description=xdg Desktop Portal
After=niri-session.target

[Service]
# ...
```

### 4.2 Update `ryoku/reload` Command

**File:** `ryoku/cli/internal/updater/reload.go`

**Change:**
```go
func Reload() error {
    // Old:
    // return exec.Command("hyprctl", "reload").Run()
    
    // New:
    return exec.Command("niri", "msg", "config-reload").Run()
}
```

### 4.3 Update `ryoku materialize`

**File:** `ryoku/cli/internal/updater/materialize.go`

**Change:**
- Remove: Copy `ryoku/hyprland/` to `~/.config/hypr/`
- Add: Copy `ryoku/niri/` to `~/.config/niri/`

### 4.4 Validation

- [ ] Full session launches in VM
- [ ] All services start correctly
- [ ] `systemctl --user status ryoku-shell` shows running
- [ ] Lockscreen works
- [ ] `ryoku reload` works

---

## Phase 5: Tool Adaptation (Days 22-25)

### 5.1 Tool Replacement Matrix

| Tool | Replace With | Status |
|------|--------------|--------|
| `hypridle` | `swayidle` | Replace |
| `hyprpicker` | Test first | Test with Niri |
| `hyprsunset` | `wlsunset` | Replace |
| `hyprpolkitagent` | `polkit-gnome` | Replace |
| `xdg-desktop-portal-hyprland` | `xdg-desktop-portal-gnome` | Replace |

### 5.2 Port Configs

**`hypridle.conf` → swayidle:**

Old:
```ini
[listener]
timeout=300
on-timeout=lock
```

New:
```bash
# swayidle config
timeout 300 'swaylock -f -c 000000'
timeout 600 'systemctl suspend'
```

**`hyprland-preview-share-picker`:** Keep (compositor-agnostic)

### 5.3 Validation

- [ ] `swayidle` works
- [ ] `hyprpicker` works (or replace with `grimblast`)
- [ ] `wlsunset` works
- [ ] `polkit-gnome` works
- [ ] Screen sharing works

---

## Phase 6: Testing (Days 26-35)

### 6.1 VM Test Matrix

**Create test VMs:**
```bash
# Single monitor
vm-curator create --name ryoku-niri-test-1 --gpu virtio --monitors 1

# Dual monitor
vm-curator create --name ryoku-niri-test-2 --gpu virtio --monitors 2

# NVIDIA passthrough (if available)
vm-curator create --name ryoku-niri-test-nvidia --gpu passthrough --nvidia

# AMD GPU (if available)
vm-curator create --name ryoku-niri-test-amd --gpu passthrough --amd
```

### 6.2 Test Checklist

**For each VM:**
- [ ] Boot to SDDM
- [ ] Select Niri session
- [ ] Login successfully
- [ ] Quickshell loads
- [ ] All keybinds work (~30 extras)
- [ ] Window management works (float, tile, resize)
- [ ] Hub UI opens and works
- [ ] Lockscreen works (Super+L)
- [ ] Wallpaper change works
- [ ] `ryoku update` works
- [ ] Bluetooth pairing (if available)
- [ ] Network (wired/WiFi)
- [ ] `niri validate` passes
- [ ] `niri msg config-reload` works
- [ ] IPC commands work

### 6.3 Critical Tests

**Config validation:**
```bash
niri validate
```

**IPC communication:**
```bash
niri msg --json focused-window
niri msg --json outputs
niri msg --json workspaces
```

**Session launch:**
```bash
# Boot to TTY, start Niri manually
niri --session
systemctl --user status ryoku-shell
systemctl --user status xdg-desktop-portal-gnome
```

### 6.4 Document Issues

- [ ] Create issue list for any bugs found
- [ ] Prioritize critical vs non-critical
- [ ] Fix critical issues before Phase 7

---

## Phase 7: Cleanup & Documentation (Days 36-40)

### 7.1 Delete Obsolete Files

**Remove directories:**
```bash
rm -rf ryoku/hyprland/
rm -rf release/packages/hypr-dynamic-cursors/
rm -rf release/packages/ryoku-hypr-plugins/
rm -rf release/packages/hyprglass/
rm -rf release/packages/imgborders/
```

**Remove individual files:**
```bash
# Any remaining Hyprland-specific files
```

### 7.2 Update Documentation

**`README.md`:**
```markdown
# Before:
"Ryoku is a hand-built Arch Linux distribution: one cohesive Hyprland desktop"

# After:
"Ryoku is a hand-built Arch Linux distribution: one cohesive Niri desktop"
```

**`docs/ryoku.md`:**
```markdown
# Before:
"The desktop is a Hyprland Wayland session authored in Lua"

# After:
"The desktop is a Niri Wayland session with config in KDL"
```

**`docs/structure.md`:**
```markdown
# Before:
- `ryoku/hyprland/` the Hyprland config, authored in Lua.

# After:
- `ryoku/niri/` the Niri config, authored in KDL.
```

**`docs/development.md`:**
- Update dev loop section
- Update config reload commands (`niri msg config-reload`)

**`docs/updates.md`:**
- Check if Niri has different update behavior

### 7.3 Update Git Hooks

**Check for Hyprland references:**
```bash
grep -r "hypr" .githooks/
```

**Update any scripts that reference Hyprland:**
- Config validation scripts
- Pre-commit hooks

### 7.4 Final Review

- [ ] All documentation updated
- [ ] No Hyprland references remain (except in `hypr.go` comments)
- [ ] All tests pass
- [ ] Code review complete
- [ ] Ready for user testing

---

## Technical Reference

### Niri Config Format (KDL)

**Location:** `~/.config/niri/config.kdl`

**Example structure:**
```kdl
// Input configuration
input {
    keyboard {
        xkb { layout "us"; }
        numlock
    }
    touchpad {
        natural-scroll
        tap
    }
}

// Output configuration
output "eDP-1" {
    mode "1920x1080@60"
    scale 1
    position x=0 y=0
}

// Layout configuration
layout {
    gaps 16
    focus-ring {
        width 4
        active-color "#7fc8ff"
        inactive-color "#505050"
    }
}

// Key bindings
binds {
    "Super+Q" = "close-window"
    "Super+Return" = { spawn = "kitty" }
}

// Spawn at startup
spawn-at-startup "gnome-keyring-daemon --start --components=secrets,pkcs11"
spawn-at-startup "qs -c shell"

// Window rules
window-rule {
    class = "kitty"
    float = true
}

// Animations (defer to Phase 2)
// animations { }
```

### Niri IPC Commands

```bash
# Get active window
niri msg --json focused-window

# Get outputs
niri msg --json outputs

# Get workspaces
niri msg --json workspaces

# Reload config
niri msg config-reload

# Validate config
niri validate

# Send action
niri msg action close-window
niri msg action spawn "kitty"
```

### Key Differences: Hyprland vs Niri

| Feature | Hyprland | Niri |
|---------|----------|------|
| Config format | Lua | KDL |
| IPC | `hyprctl` | `niri msg` |
| Workspaces | Discrete (1-10) | Scrollable strip |
| Keybind reload | `hyprctl reload` | `niri msg config-reload` |
| Plugin system | Yes (Hyprland plugins) | No (or minimal) |
| Animations | Customizable | Built-in, less customizable |
| Multi-monitor | Shared workspace space | Independent per monitor |

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Niri IPC doesn't support a needed feature | High | Check niri-ipc crate docs, file issue if missing |
| Config KDL syntax is complex | Medium | Start with minimal config, incrementally add |
| Multi-monitor model differs significantly | Medium | Test early, document differences |
| Quickshell doesn't work with Niri | High | Test immediately in Phase 1 |
| Screen recording doesn't work | Medium | Fall back to wf-recorder or pipewire-native |
| NVIDIA drivers have issues | Medium | Follow Niri NVIDIA wiki page, test early |
| Workspace model change confuses users | Medium | Document differences, provide migration guide |

---

## Success Criteria

1. ✅ Fresh install works: ISO boots → Niri → Ryoku desktop
2. ✅ Hub manages Niri: All Hub pages work with Niri backend
3. ✅ Keybinds ported: All ~30 extra keybinds from Hyprland → Niri
4. ✅ Shell works: Quickshell desktop runs under Niri
5. ✅ Lockscreen works: In-session lock and SDDM login
6. ✅ Updates work: `ryoku update` manages Niri configs
7. ✅ Performance: Comparable or better than Hyprland
8. ✅ Documentation: Users understand the change and how to use it

---

## Implementation Checklist

### Phase 1: Packages (Days 1-3)
- [ ] Update `system/packages/base.packages`
- [ ] Update `release/packages/ryoku-desktop/PKGBUILD`
- [ ] Delete Hyprland plugin packages
- [ ] Test: Package builds, dependencies resolve

### Phase 2: Config (Days 4-10)
- [ ] Create `ryoku/niri/` directory
- [ ] Create `ryoku/niri/config.kdl`
- [ ] Port keyboard, monitors, input, decoration
- [ ] Port binds (extras only)
- [ ] Port window rules
- [ ] Port autostart (systemd services)
- [ ] Test: `niri validate` passes
- [ ] Test: Niri starts in VM

### Phase 3: Hub Backend (Days 11-17)
- [ ] Create `ryoku/hub/backend/niri.go`
- [ ] Implement `GenerateConfig()`
- [ ] Implement `ReloadConfig()`
- [ ] Implement IPC functions
- [ ] Update `main.go` to use `niri.go`
- [ ] Test: Hub UI works with Niri

### Phase 4: Session (Days 18-21)
- [ ] Create systemd user units
- [ ] Update `ryoku/reload` command
- [ ] Update `ryoku materialize`
- [ ] Test: Full session launches

### Phase 5: Tools (Days 22-25)
- [ ] Replace `hypridle` → `swayidle`
- [ ] Test `hyprpicker` (keep or replace)
- [ ] Replace `hyprsunset` → `wlsunset`
- [ ] Replace `hyprpolkitagent` → `polkit-gnome`
- [ ] Test: All tools work

### Phase 6: Testing (Days 26-35)
- [ ] Create test VMs
- [ ] Run full test matrix
- [ ] Document issues
- [ ] Fix critical issues

### Phase 7: Cleanup (Days 36-40)
- [ ] Delete `ryoku/hyprland/`
- [ ] Update documentation
- [ ] Update git hooks
- [ ] Final review

---

## References

- **Niri Documentation:** https://niri-wm.github.io/niri/
- **Niri Config:** https://niri-wm.github.io/niri/Configuration:-Introduction.html
- **Niri IPC:** https://niri-wm.github.io/niri/IPC.html
- **Niri GitHub:** https://github.com/niri-wm/niri
- **DankMaterialShell (Niri-compatible shell):** https://github.com/AvengeMedia/DankMaterialShell

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-08-28 | Initial migration plan |

---

## Notes for Future Agents

- **This is a clean break migration** - no dual support, no rollback
- **`hypr.go` is kept as reference** but not used in code
- **Keybinds are partial** - only port extras, not window management
- **Animations are deferred** - Phase 2 work
- **Testing is critical** - use QEMU VMs with vm-curator
- **Niri's workspace model is different** - scrollable strip vs discrete workspaces
- **Hub backend needs full rewrite** - KDL output, `niri msg` IPC
- **Autostart should use systemd** - better dependency management than `spawn-at-startup`