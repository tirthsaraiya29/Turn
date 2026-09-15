# Turn

**Event-driven workspace automation and context synchronization for Windows and Android.**

Turn is a local-first automation daemon for Windows that reacts to real operating-system and hardware events instead of continuously polling for changes.

It lets you define context-aware modes such as:

* When my laptop is plugged in and my monitor is connected → enter **Desk Mode**
* When I launch a game → switch performance and audio settings
* When I disconnect from my home network → leave **Home Mode**
* When my phone is nearby and my PC is docked → synchronize the workspace
* When a condition stops being true → restore the state that existed before the mode started

Turn is built around four ideas:

> **Listen. Decide. Transform. Restore.**

It does not require a cloud service, automation broker, or permanently running scripting environment.

> **Development Status: Idea**

---

## Why Turn?

Most desktop automation falls into one of two categories.

### Manual Automation

You create shortcuts, profiles, or modes and activate them yourself.

### Polling Automation

A program repeatedly checks whether something changed:

```text
while running:
    check_power()
    check_network()
    check_display()
    check_processes()
    sleep(...)
```

Turn takes a different approach:

```text
Windows / Hardware
       │
       │ native events
       ▼
   Turn Runtime
       │
       ▼
Context Evaluation
       │
       ▼
    Enter Mode
       │
       ▼
Transform System
       │
       ▼
Condition Exits
       │
       ▼
 Restore State
```

Where Windows provides a suitable event mechanism, Turn prefers it over polling.

---

# Quick Start

Turn is currently an idea, so there is no working release to install yet.

The intended first workflow is:

```text
1. Install Turn
      ↓
2. Open the Turn UI
      ↓
3. Create a recipe
      ↓
4. Select triggers
      ↓
5. Select actions
      ↓
6. Save the recipe
      ↓
7. Turn listens for native Windows events
      ↓
8. Matching context activates the mode
      ↓
9. Turn applies the actions
      ↓
10. When the context disappears,
    Turn restores the previous state
```

### Example

```text
Monitor connected
        +
Home Wi-Fi
        +
AC power
        │
        ▼
   "Desk Mode"
        │
        ├── Set display to 144 Hz
        ├── Switch audio output
        ├── Enable focus settings
        └── Sync phone
```

The minimal configuration would look conceptually like:

```json
{
  "name": "Desk Mode",

  "conditions": {
    "all": [
      {
        "type": "display.connected",
        "monitor": "DEL41A8"
      },
      {
        "type": "network.wifi",
        "ssid": "Home-5G"
      },
      {
        "type": "power.source",
        "value": "ac"
      }
    ]
  },

  "actions": [
    {
      "type": "windows.display.refresh_rate",
      "value": 144
    },
    {
      "type": "windows.audio.default_endpoint",
      "device_id": "MY_DAC"
    }
  ]
}
```

---

# Features

## Event-Driven Triggers

Turn is designed to consume events from Windows subsystems rather than repeatedly checking system state.

### Power

* AC / battery transitions
* Charging state
* Battery percentage
* Power-source changes

### Network

* Wi-Fi network changes
* SSID changes
* Network adapter changes
* Ethernet link changes
* VPN interface changes
* Network availability

### Display

* Monitor connection/disconnection
* Display topology changes
* Monitor identity / EDID information
* Resolution changes
* Refresh-rate changes
* HDR-related state where exposed by the platform

### Audio

* Default audio endpoint changes
* Audio device availability
* Audio-session changes
* Microphone activity
* USB audio device connection

### Devices

* USB device arrival/removal
* Device identification using VID/PID
* Selected hardware state changes

### Processes & Workspace

* Foreground application changes
* Process creation/termination
* Full-screen application state where detectable
* Virtual desktop/workspace changes where supported

### Android Context

When paired with the Android companion, Turn can consume context that Windows cannot observe directly, including:

* Battery state
* Device connection state
* Orientation
* Proximity information
* Incoming-call state where permitted
* Other events exposed through Android APIs and granted permissions

---

# Actions

A trigger describes **when something happens**.

An action describes **what Turn does**.

Turn's Windows enforcers are modular so new system integrations can be added without changing the core automation engine.

## Audio

* Change default audio endpoint
* Adjust master volume
* Mute/unmute microphone
* Apply supported per-application audio changes

## Performance

* Change Windows power scheme
* Adjust process power throttling
* Set process CPU affinity
* Apply foreground/background performance policies

## Display

* Change supported display configuration
* Change refresh rate
* Apply display-mode changes

## Workspace

* Move windows
* Apply window layouts
* Change supported virtual-desktop state
* Apply supported Windows theme settings

## Notifications

* Apply supported Windows notification and focus settings

## Security & Networking

* Activate/deactivate predefined firewall rules
* Flush DNS
* Lock the workstation
* Clear the clipboard

## Android

Where Android permits the operation through its public APIs and granted permissions:

* Synchronize notification state
* Trigger haptic confirmation
* Adjust supported audio states
* Exchange battery/device context
* Synchronize workspace modes

> **Important:** Turn does not bypass Windows or Android security boundaries. Some operations require elevation, explicit permissions, or are restricted by the operating system.

---

# Modes, Not Scripts

Turn recipes describe **desired state transitions**, rather than arbitrary programs.

For example:

```text
Desk detected
    │
    ├── Monitor connected
    ├── Home Wi-Fi connected
    └── AC power
          │
          ▼
      Desk Mode
          │
          ├── 144 Hz display
          ├── DAC as default output
          ├── Focus configuration
          └── Android synchronization
```

When the desk context disappears:

```text
Desk Mode
    │
    ▼
Context no longer valid
    │
    ▼
Restore captured state
```

The important part is that Turn does not assume what the previous configuration was.

---

# Deterministic State & Rollback

A common problem with automation is **configuration drift**.

Suppose the system starts as:

```text
Volume       = 73
Power Plan   = Balanced
Refresh Rate = 60 Hz
```

Turn enters a mode:

```text
Volume       = 35
Power Plan   = High Performance
Refresh Rate = 144 Hz
```

A naive automation system might restore hard-coded values:

```text
Volume       → 100
Power Plan   → Balanced
Refresh Rate → 60 Hz
```

But this can overwrite changes made by the user or another automation.

Turn instead records the state it actually replaced:

```text
Before Mode
    │
    ├── Volume       = 73
    ├── Power Plan   = Balanced
    └── Refresh Rate = 60 Hz
    │
    ▼
Capture Snapshot
    │
    ▼
Apply Mode
```

When the mode exits, Turn restores the captured state, subject to its state-ownership and conflict rules.

---

# State Ownership

Rollback becomes complicated when multiple modes overlap.

For example:

```text
Normal
   │
   ▼
Desk Mode
   │
   ▼
Gaming Mode
```

Both modes might modify the same properties.

Turn therefore treats system mutations as **owned state changes**, rather than simply maintaining a list of "undo" commands.

Conceptually:

```text
Property
   │
   ├── Current value
   ├── Owning mode
   ├── Previous value
   └── Restoration metadata
```

This allows Turn to reason about:

* Mode priority
* Overlapping modes
* Conflicting actions
* Mode exit order
* External user changes
* Failed actions
* Partial rollback

The goal is to prevent one automation from blindly undoing another automation's changes.

---

# Conditions

Recipes can combine multiple conditions.

Supported logical operators include:

* `all`
* `any`
* `not`

For example:

```text
IF

    monitor == connected
AND
    wifi == "Home-5G"
AND
    power == AC

THEN

    activate Desk Mode
```

Conditions can also use:

* Debounce
* Hysteresis
* Thresholds
* Priority
* Entry/exit semantics

This prevents rapidly changing hardware state from causing mode flapping.

---

# Example Recipe

```json
{
  "$schema": "./schema/recipe.v1.json",
  "name": "Deep Work & Desk Docked",
  "priority": 80,
  "debounce_ms": 1500,

  "conditions": {
    "all": [
      {
        "type": "display.connected",
        "monitor": {
          "edid": "DEL41A8"
        }
      },
      {
        "type": "network.wifi",
        "operator": "equals",
        "ssid": "Home-5G"
      },
      {
        "type": "power.source",
        "operator": "equals",
        "value": "ac"
      }
    ]
  },

  "actions": [
    {
      "type": "windows.audio.default_endpoint",
      "device_id": "{0.0.0.00000000}.{usb_audio_dac_guid}",
      "rollback": true
    },
    {
      "type": "windows.display.refresh_rate",
      "value": 144,
      "rollback": true
    },
    {
      "type": "windows.notifications.focus",
      "value": "priority_only",
      "rollback": true
    },
    {
      "type": "android.sync",
      "operation": "focus_mode",
      "payload": {
        "enabled": true
      }
    }
  ]
}
```

Recipes are intended to live locally under:

```text
%USERPROFILE%\.turn\
├── recipes.json
├── state.json
└── identity\
```

The exact on-disk state format is implementation-defined and should not be modified manually unless documented by the project.

---

# Recipe Lifecycle

Every recipe moves through a predictable lifecycle:

```text
             ┌─────────────┐
             │   INACTIVE  │
             └──────┬──────┘
                    │
                OS event
                    │
                    ▼
          ┌──────────────────┐
          │ Evaluate Context │
          └────────┬─────────┘
                   │
             Conditions?
              ┌────┴────┐
             No         Yes
             │           │
             │           ▼
             │      Debounce /
             │      Hysteresis
             │           │
             │           ▼
             │      Priority Check
             │           │
             │           ▼
             │      Capture State
             │           │
             │           ▼
             │      Apply Actions
             │           │
             │           ▼
             │       ┌────────┐
             └──────▶│ ACTIVE │
                     └───┬────┘
                         │
                   Exit condition
                         │
                         ▼
                   Restore State
                         │
                         ▼
                     INACTIVE
```

---

# Architecture

Turn is divided into independent layers.

```text
┌─────────────────────────────────────────────────────────────┐
│                         TURN UI                             │
│                  WinUI 3 / System Tray                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      TURN RUNTIME                           │
│                                                             │
│ Event Bus → Condition Engine → Priority Arbiter             │
│                         │                                   │
│                         ▼                                   │
│                State / Ownership Manager                    │
│                         │                                   │
│                         ▼                                   │
│                    Action Engine                            │
└───────────────┬─────────────────────────┬───────────────────┘
                │                         │
                ▼                         ▼
┌──────────────────────────┐   ┌─────────────────────────────┐
│    WINDOWS ENFORCERS     │   │       MOBILE RELAY          │
│                          │   │                             │
│ Audio                    │   │ Discovery                   │
│ Display                  │   │ Pairing                     │
│ Power                    │   │ Transport                   │
│ QoS                      │   │ Android state               │
│ Workspace                │   │                             │
│ Network                  │   │                             │
└──────────────────────────┘   └──────────────┬──────────────┘
                                              │
                                              │ Local P2P
                                              ▼
                                   ┌──────────────────────┐
                                   │ Android Companion    │
                                   └──────────────────────┘

                    ┌───────────────────────────┐
                    │    OPTIONAL PARSER        │
                    │                           │
                    │ Natural Language          │
                    │       ↓                   │
                    │ Local Model               │
                    │       ↓                   │
                    │ Recipe Compiler           │
                    │       ↓                   │
                    │ Schema Validation         │
                    └───────────────────────────┘
```

---

# Project Structure

```text
Turn/
│
├── src/
│   │
│   ├── Turn.Daemon/
│   │   ├── Core/
│   │   │   ├── Runtime/
│   │   │   ├── State/
│   │   │   ├── Ownership/
│   │   │   ├── Conditions/
│   │   │   └── Arbitration/
│   │   │
│   │   ├── Events/
│   │   │   ├── Power/
│   │   │   ├── Network/
│   │   │   ├── Display/
│   │   │   ├── Audio/
│   │   │   ├── Devices/
│   │   │   └── Processes/
│   │   │
│   │   ├── Enforcers/
│   │   │   ├── Audio/
│   │   │   ├── Display/
│   │   │   ├── Power/
│   │   │   ├── QoS/
│   │   │   ├── Workspace/
│   │   │   ├── Notifications/
│   │   │   └── Network/
│   │   │
│   │   └── Relay/
│   │       ├── Discovery/
│   │       ├── Pairing/
│   │       ├── Transport/
│   │       └── Protocol/
│   │
│   ├── Turn.Mobile/
│   │   └── app/
│   │       └── src/
│   │           └── main/
│   │               └── kotlin/
│   │
│   ├── Turn.Parser/
│   │   ├── Compiler/
│   │   ├── Validation/
│   │   └── Model/
│   │
│   └── Turn.UI/
│       ├── Views/
│       ├── ViewModels/
│       ├── Tray/
│       └── Settings/
│
├── schema/
│   └── recipe.v1.json
│
├── tests/
│   ├── Core/
│   ├── Events/
│   ├── Enforcers/
│   ├── Relay/
│   ├── Recipes/
│   ├── Integration/
│   ├── Mock/
│   └── Fuzz/
│
├── docs/
├── LICENSE
└── README.md
```

---

# Event Sources

Turn is designed to use native Windows mechanisms where possible.

| Event             | Possible Windows mechanism                         |
| ----------------- | -------------------------------------------------- |
| Power changes     | Power setting notifications / `WM_POWERBROADCAST`  |
| Device arrival    | `WM_DEVICECHANGE`                                  |
| Foreground window | WinEvent hooks                                     |
| Display changes   | Display configuration APIs / display notifications |
| Audio changes     | Windows Core Audio notifications                   |
| Process events    | ETW / Windows process notifications                |
| Network changes   | Windows networking APIs                            |
| System telemetry  | ETW where appropriate                              |

The implementation should use the narrowest reliable event source available instead of introducing polling simply for convenience.

---

# Windows Integration

Turn is intended to operate primarily in user space.

The core automation engine does not require a kernel driver.

Some individual integrations may require:

* Administrator privileges
* Specific Windows capabilities
* User consent
* Access to restricted interfaces
* Windows-specific APIs

Platform-specific integrations remain isolated inside their respective event providers and enforcers.

For example:

```text
Turn Runtime
     │
     ▼
Audio Enforcer Interface
     │
     ├── Core Audio implementation
     └── Windows-specific endpoint implementation
```

The core runtime does not need to know how Windows performs the underlying mutation.

---

# Android Companion

The Android application acts as Turn's mobile context provider and remote endpoint.

```text
┌─────────────────┐             ┌─────────────────┐
│  Turn Windows   │◄───────────►│  Turn Android   │
└─────────────────┘   Local     └─────────────────┘
                      P2P
```

The companion can provide context that Windows cannot observe directly.

For example:

```text
Phone battery < 20%
        │
        ▼
Android Companion
        │
        ▼
Encrypted Local Connection
        │
        ▼
Turn
        │
        ▼
Battery-saving workstation mode
```

Android functionality is permission-aware and does not attempt to circumvent Android's security model.

---

# Local Pairing

Turn is designed to work without a cloud account.

The intended pairing flow is:

```text
Windows                         Android
   │                               │
   │◄──── Local discovery ─────────│
   │                               │
   │──── Pairing request ─────────►│
   │                               │
   │◄──── Authentication data ─────│
   │                               │
   │──── User verifies SAS ───────►│
   │                               │
   │◄════ Authenticated channel ══►│
```

The implementation should establish a persistent device identity and authenticate subsequent connections rather than trusting devices solely because they are present on the local network.

### Potential Transports

* Local Wi-Fi
* TCP
* TLS
* BLE GATT fallback

Cryptographic implementation should use established libraries rather than custom cryptographic primitives.

---

# Privacy

Turn follows a **local-first** model.

By default, the architecture requires:

* No cloud account
* No remote automation server
* No external broker
* No telemetry
* Local recipe storage
* Local context processing
* Direct device-to-device synchronization

Internet access should not be required for ordinary automation.

---

# Local Recipe Compiler

Turn can optionally include a small local model that translates natural-language descriptions into recipes.

For example:

> "Whenever I connect my monitor at home while charging, turn on deep work mode."

The intended pipeline is:

```text
Natural Language
       ↓
Local Model
       ↓
Structured Recipe
       ↓
Schema Validation
       ↓
Semantic Validation
       ↓
Permission Analysis
       ↓
Conflict Analysis
       ↓
User Approval
       ↓
Installation
```

The model is not given unrestricted authority over the machine.

The model translates intent; Turn's runtime remains responsible for execution.

---

# Testing

Turn is intended to include a hardware-independent test environment.

The goal is to test the automation engine without modifying the real system.

Example:

```powershell
Turn.Daemon.exe `
    --test-harness `
    --recipe "Deep Work & Desk Docked" `
    --simulate "ac_disconnect"
```

A simulated event should travel through the same logical runtime pipeline:

```text
Simulated Event
      ↓
Event Bus
      ↓
Condition Engine
      ↓
Priority Arbiter
      ↓
State Manager
      ↓
Mock Enforcer
      ↓
Assertions
      ↓
Rollback Verification
```

Tests should cover:

* Condition evaluation
* Compound conditions
* Debouncing
* Hysteresis
* Priority conflicts
* State snapshots
* State ownership
* Mode entry
* Mode exit
* Rollback
* Failed actions
* Partial execution
* Device disappearance
* Reordered events
* Duplicate events
* Invalid recipes
* Concurrent modes

---

# Fuzz Testing

Recipes and event streams should be treated as untrusted input.

Important fuzzing targets include:

```text
Recipe Parser
    ├── Malformed JSON
    ├── Unknown condition types
    ├── Invalid operators
    ├── Deeply nested expressions
    ├── Oversized payloads
    └── Invalid action parameters

Runtime
    ├── Duplicate events
    ├── Event storms
    ├── Reordered events
    ├── Device disappearance
    ├── Failed actions
    └── Rapid enter/exit cycles
```

A malformed recipe must never result in arbitrary system modification.

---

# Requirements

## Windows

* Windows 10 version 2004 / build 19041 or newer
* Windows 11
* Windows SDK 10.0.22621 or newer
* .NET 8 SDK
* Visual Studio 2022 for native Windows components

## Android

* Android 8.0 / API 26 or newer
* Bluetooth LE support for BLE transport
* Permissions required by the specific Android features enabled

---

# Building

> **Note:** Turn is currently an idea. The commands below describe the intended development setup and are not yet a promise of a working build.

## Clone

```bash
git clone https://github.com/username/Turn.git
cd Turn
```

## Build the Daemon

```bash
dotnet build Turn.sln -c Release
```

## Self-Contained Windows Build

```powershell
dotnet publish `
    src/Turn.Daemon/Turn.Daemon.csproj `
    -c Release `
    -r win-x64 `
    --self-contained true
```

## Build Android

```bash
cd src/Turn.Mobile
./gradlew assembleDebug
```

## Install a Debug Build

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```

---

# Development

The intended development commands are:

## Run the Daemon

```powershell
Turn.Daemon.exe --run
```

## Diagnostics

```powershell
Turn.Daemon.exe --diagnostics
```

## Recipe Simulation

```powershell
Turn.Daemon.exe `
    --test-harness `
    --recipe "Deep Work & Desk Docked" `
    --simulate "ac_connect"
```

> These commands describe the planned development interface and may not currently exist.

---

# Design Principles

## 1. Events Over Polling

If the operating system can tell us that something happened, listen to it.

## 2. State Over Commands

A mode represents a desired state, not a collection of irreversible commands.

## 3. Capture Before Mutation

Do not assume what the system looked like before Turn changed it.

## 4. Explicit Ownership

Multiple automations must be able to coexist without blindly undoing each other's changes.

## 5. Local by Default

Core automation should work without an internet connection.

## 6. Least Privilege

A feature should not require administrator access merely because another feature does.

## 7. Models Do Not Get System Authority

Natural-language generation may create a recipe, but validation and explicit policy remain responsible for execution.

## 8. Deterministic Execution

Given the same initial state, event sequence, and recipe set, the runtime should produce predictable results.

## 9. Failure Is a State

Actions can fail. Devices can disappear. Processes can terminate.

Turn should model those situations rather than assuming every action succeeds.

---

# Roadmap

## Phase 1 — Core Runtime

* [ ] Define recipe schema
* [ ] Build event bus
* [ ] Build condition engine
* [ ] Add debounce
* [ ] Add hysteresis
* [ ] Implement priority arbitration
* [ ] Implement state ownership
* [ ] Implement snapshot manager
* [ ] Implement rollback engine
* [ ] Build mock event bus

## Phase 2 — Windows

* [ ] Power events
* [ ] Network events
* [ ] Display events
* [ ] Audio events
* [ ] USB/device events
* [ ] Process events
* [ ] Foreground-window events
* [ ] Audio enforcer
* [ ] Power-plan enforcer
* [ ] Display enforcer
* [ ] QoS enforcer
* [ ] Workspace enforcer
* [ ] Notification enforcer
* [ ] Network enforcer

## Phase 3 — Android

* [ ] Companion application
* [ ] Local discovery
* [ ] Device pairing
* [ ] Authenticated transport
* [ ] BLE fallback
* [ ] Battery context
* [ ] Orientation context
* [ ] Proximity context
* [ ] Notification synchronization

## Phase 4 — UI

* [ ] System tray
* [ ] Recipe editor
* [ ] Active modes
* [ ] Event timeline
* [ ] State inspector
* [ ] Permission manager
* [ ] Device pairing
* [ ] Diagnostics

## Phase 5 — Local Intelligence

* [ ] Local recipe compiler
* [ ] ONNX inference
* [ ] Recipe generation
* [ ] Semantic validation
* [ ] Permission analysis
* [ ] Conflict analysis
* [ ] Natural-language recipe repair

---

# Contributing

Turn is currently an idea, so the architecture and interfaces are expected to change substantially.

When development begins, contributions should follow these principles:

1. Keep platform-specific code inside event providers or enforcers.
2. Prefer native event notifications over polling.
3. Add mock implementations where practical.
4. Define rollback behavior for reversible state mutations.
5. Handle partial failure explicitly.
6. Validate all recipe input.
7. Add tests for entry, active, exit, and failure states.
8. Document required permissions and privileges.
9. Do not introduce cloud dependencies into the core runtime.
10. Do not implement custom cryptographic primitives.

---

# License

Turn is intended to be open-source software.

The project license will be defined in [`LICENSE`](LICENSE).

---

# Development Status

> **Status: Idea**

Turn is currently a project concept and architecture rather than a production-ready application.

The APIs, recipe schema, transport protocol, project structure, and implementation details described in this README are subject to change as the project is prototyped and validated.

---

# The Idea

```text
                    TURN

         Observe the environment
                    │
                    ▼
           Understand context
                    │
                    ▼
             Enter the mode
                    │
                    ▼
          Transform the system
                    │
                    ▼
          Context disappears
                    │
                    ▼
         Restore previous state
```

Turn makes your devices respond to what you're doing and where you are — **without requiring the cloud to be involved.**

---

## Listen. Decide. Transform. Restore.
