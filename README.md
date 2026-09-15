Turn

Event-driven workspace automation and context synchronization for Windows and Android.

Turn is a local-first automation daemon for Windows that reacts to real operating-system and hardware events instead of continuously polling for changes.

It lets you define context-aware modes such as:

- When my laptop is plugged in and my monitor is connected → enter desk mode.
- When I launch a game → switch performance settings and audio.
- When I disconnect from my home network → leave home mode.
- When my phone is nearby and my PC is docked → synchronize the workspace.
- When a condition stops being true → restore the state that existed before the mode started.

Turn is designed around four ideas:

«Listen. Decide. Transform. Restore.»

It does not require a cloud service, automation broker, or permanently running scripting environment.

---

Why Turn?

Most desktop automation falls into one of two categories:

Manual automation

You create shortcuts, profiles, or modes and activate them yourself.

User
 │
 ├── Open application
 ├── Change power mode
 ├── Change audio device
 ├── Enable DND
 └── Change display settings

Polling automation

A program repeatedly checks whether something changed.

while running:
    check_power()
    check_network()
    check_display()
    check_processes()
    sleep(...)

Turn takes a different approach.

              Windows / Hardware
                     │
                     │ native events
                     ▼
              ┌──────────────┐
              │ Turn Runtime │
              └──────┬───────┘
                     │
              evaluate context
                     │
                     ▼
                enter mode
                     │
                     ▼
              change the system
                     │
              condition exits
                     │
                     ▼
               restore state

Where Windows provides a suitable event mechanism, Turn prefers it over polling.

---

Features

Event-driven triggers

Turn can consume events from several Windows subsystems.

Power

- AC / battery transitions
- Charging state
- Battery percentage
- Power-source changes

Network

- Wi-Fi network changes
- SSID changes
- Network adapter changes
- Ethernet link changes
- VPN interface changes
- Network availability

Display

- Monitor connection/disconnection
- Display topology changes
- Monitor identity / EDID information
- Resolution and refresh-rate changes
- HDR-related display state where exposed by the platform

Audio

- Default audio endpoint changes
- Audio device availability
- Audio-session changes
- Microphone activity
- USB audio device connection

Devices

- USB device arrival/removal
- Device identification using VID/PID
- Selected hardware state changes

Processes & workspace

- Foreground application changes
- Process creation/termination
- Full-screen application state where detectable
- Virtual desktop/workspace changes where supported

Android context

When paired with the Android companion, Turn can additionally consume device context such as:

- Battery state
- Device connection state
- Orientation
- Proximity information
- Incoming-call state where permitted
- Other Android events exposed through the companion's permissions

---

Actions

A trigger only describes when something happens.

An action describes what Turn does.

Turn's Windows enforcers are modular so that new system integrations can be added without changing the core automation engine.

Possible actions include:

Audio

- Change default audio endpoint
- Adjust master volume
- Mute/unmute microphone
- Apply supported per-application audio changes

Performance

- Change Windows power scheme
- Adjust process power throttling
- Set process CPU affinity
- Apply foreground/background performance policies

Display

- Change supported display configuration
- Change refresh rate
- Apply display-mode changes

Workspace

- Move windows
- Apply window layouts
- Change supported virtual-desktop state
- Apply Windows theme-related settings

Notifications

- Apply supported Windows notification / focus settings

Security & networking

- Activate/deactivate predefined firewall rules
- Flush DNS
- Lock the workstation
- Clear the clipboard

Android

Where Android permits the operation through its public APIs and granted permissions:

- Synchronize notification state
- Trigger a haptic confirmation
- Adjust supported audio states
- Exchange battery/device context
- Synchronize workspace modes

«Important: Turn does not bypass Windows or Android security boundaries. Some operations require elevation, explicit permissions, or are restricted by the operating system.»

---

Modes, Not Scripts

Turn recipes describe desired state transitions, rather than arbitrary programs.

For example:

Desk detected
    │
    ├── Monitor connected
    ├── Home Wi-Fi connected
    └── AC power
          │
          ▼
     "Desk Mode"
          │
          ├── 144 Hz display
          ├── DAC as default output
          ├── Focus configuration
          └── Android synchronization

When the desk context disappears:

Desk Mode
    │
    ▼
Context no longer valid
    │
    ▼
Restore captured state

The important part is that Turn does not assume what the previous configuration was.

---

Deterministic State & Rollback

A common problem with automation is configuration drift.

Imagine:

Before automation:

Volume = 73
Power Plan = Balanced
Refresh Rate = 60 Hz

Turn enters a mode:

Volume = 35
Power Plan = High Performance
Refresh Rate = 144 Hz

The mode later exits.

A naive automation system might do:

Volume → 100
Power Plan → Balanced
Refresh Rate → 60 Hz

But the user may have changed the volume to "61" while the mode was active.

Turn instead records the state it actually replaced.

┌─────────────────────────┐
│ Original State           │
│                         │
│ Volume       73          │
│ Power Plan   Balanced    │
│ Refresh      60 Hz       │
└────────────┬────────────┘
             │
             │ snapshot
             ▼
┌─────────────────────────┐
│ Active Mode              │
│                         │
│ Volume       35          │
│ Power Plan   High        │
│ Refresh      144 Hz      │
└─────────────────────────┘

On exit, Turn restores the captured state subject to its conflict and ownership rules.

---

State Ownership

Rollback becomes complicated when multiple modes overlap.

For example:

Normal
  │
  ▼
Desk Mode
  │
  ▼
Gaming Mode

Both modes may modify volume.

Turn therefore treats system mutations as owned state changes, rather than simply maintaining a list of arbitrary "undo" commands.

Conceptually:

Property
   │
   ├── current value
   ├── owning mode
   ├── previous value
   └── restoration metadata

This allows Turn to reason about:

- Mode priority
- Overlapping modes
- Conflicting actions
- Mode exit order
- External user changes
- Failed actions
- Partial rollback

The goal is to prevent one automation from blindly undoing another automation's work.

---

Conditions

Recipes can combine multiple conditions.

Supported logical operators include:

all
any
not

A recipe can therefore express:

IF

    monitor == connected
AND
    wifi == "Home-5G"
AND
    power == AC

THEN

    activate Desk Mode

Conditions can also use:

- Debounce
- Hysteresis
- Thresholds
- Priority
- Entry/exit semantics

This prevents rapidly changing hardware state from causing mode flapping.

---

Example Recipe

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

Recipes live locally:

%USERPROFILE%\.turn\
├── recipes.json
├── state.json
└── identity\

The exact on-disk state format is implementation-defined and should not be modified manually unless documented by the project.

---

Recipe Lifecycle

Every active recipe moves through a state machine.

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
                    conditions?
                    ┌─────┴─────┐
                   no           yes
                   │             │
                   │             ▼
                   │       Debounce /
                   │       Hysteresis
                   │             │
                   │             ▼
                   │       Priority Check
                   │             │
                   │             ▼
                   │       Capture State
                   │             │
                   │             ▼
                   │       Apply Actions
                   │             │
                   │             ▼
                   │        ┌─────────┐
                   └───────▶│ ACTIVE  │
                            └────┬────┘
                                 │
                          exit condition
                                 │
                                 ▼
                         Restore State
                                 │
                                 ▼
                            INACTIVE

---

Architecture

Turn is divided into independent layers.

┌──────────────────────────────────────────────────────────────┐
│                         TURN UI                              │
│                  WinUI 3 / System Tray                      │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                       TURN RUNTIME                           │
│                                                              │
│  Event Bus → Condition Engine → Priority Arbiter             │
│                         │                                    │
│                         ▼                                    │
│                State / Ownership Manager                     │
│                         │                                    │
│                         ▼                                    │
│                    Action Engine                             │
└───────────────┬─────────────────────────┬────────────────────┘
                │                         │
                ▼                         ▼
┌──────────────────────────┐   ┌──────────────────────────────┐
│   WINDOWS ENFORCERS      │   │       MOBILE RELAY           │
│                          │   │                              │
│ Audio                    │   │ Discovery                    │
│ Display                  │   │ Pairing                      │
│ Power                    │   │ Transport                    │
│ QoS                      │   │ Android state                │
│ Workspace                │   │                              │
│ Network                  │   │                              │
└──────────────────────────┘   └──────────────┬───────────────┘
                                               │
                                               │ local P2P
                                               ▼
                                    ┌─────────────────────┐
                                    │ Android Companion   │
                                    └─────────────────────┘

             ┌──────────────────────────────┐
             │     OPTIONAL PARSER          │
             │                              │
             │ Natural Language             │
             │       ↓                      │
             │ Local Model                  │
             │       ↓                      │
             │ Recipe Compiler              │
             │       ↓                      │
             │ Schema + Semantic Validation │
             └──────────────────────────────┘

---

Project Structure

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
│
├── LICENSE
└── README.md

---

Event Sources

Turn intentionally uses native Windows mechanisms where possible.

Examples include:

Event| Possible Windows mechanism
Power changes| Power setting notifications / "WM_POWERBROADCAST"
Device arrival| "WM_DEVICECHANGE"
Foreground window| WinEvent hooks
Display changes| Display configuration APIs / display notifications
Audio changes| Windows Core Audio notifications
Process events| ETW / Windows process notifications
Network changes| Windows networking APIs
System telemetry| ETW where appropriate

The implementation should use the narrowest reliable event source available instead of introducing a polling loop simply for convenience.

---

Windows Integration

Turn is intended to remain a normal user-space application.

It does not require a kernel driver for its core automation functionality.

Some operations may require:

- Administrator privileges
- Specific Windows capabilities
- User consent
- Access to interfaces that are restricted or undocumented

Platform-specific integrations should remain isolated inside the relevant enforcer.

For example:

Turn Runtime
     │
     ▼
Audio Enforcer Interface
     │
     ├── Core Audio implementation
     └── Windows-specific endpoint implementation

The core engine should not need to know how Windows changes the audio endpoint.

---

Android Companion

The Android application acts as Turn's mobile context provider and remote endpoint.

                    Local Network
┌─────────────────┐                 ┌─────────────────┐
│                 │                 │                 │
│  Turn Windows   │◄───────────────►│  Turn Android   │
│                 │                 │                 │
└─────────────────┘                 └─────────────────┘

The companion can provide context that Windows cannot observe directly.

For example:

Phone battery < 20%
        │
        ▼
Android Companion
        │
        ▼
Encrypted local connection
        │
        ▼
Turn
        │
        ▼
Activate battery-saving workstation mode

Android functionality is deliberately permission-aware. Turn does not attempt to circumvent Android's security model.

---

Local Pairing

Turn is designed to operate without a cloud account.

Initial pairing can use:

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

The implementation should establish a persistent device identity and authenticate subsequent connections rather than trusting devices solely because they are present on the local network.

The transport may use:

- Local Wi-Fi
- TCP
- TLS
- BLE GATT fallback

Cryptographic implementation details belong to the protocol specification and should use established libraries rather than custom cryptography.

---

Privacy

Turn follows a local-first model.

By default:

- No cloud account
- No remote automation server
- No telemetry
- No third-party broker
- Recipes remain local
- Context processing occurs locally
- Mobile synchronization happens directly between paired devices

Internet access should not be required for ordinary automation.

---

Local Recipe Compiler

Turn can optionally include a small local model for generating recipes from natural language.

Example:

"Whenever I connect my monitor at home while charging,
turn on deep work mode."

                    │
                    ▼
             Local model
                    │
                    ▼
             Recipe compiler
                    │
                    ▼
             JSON Schema check
                    │
                    ▼
           Semantic validation
                    │
                    ▼
              User approval
                    │
                    ▼
                Recipe

The model is not given unrestricted authority over the machine.

Generated recipes must pass validation before installation.

Recommended pipeline:

Natural language
       ↓
Local model
       ↓
Structured recipe
       ↓
Schema validation
       ↓
Semantic validation
       ↓
Permission analysis
       ↓
Conflict analysis
       ↓
User approval
       ↓
Installation

This keeps the model responsible for translation, not system authority.

---

Testing

Turn includes a hardware-independent test environment.

The goal is to test the automation engine without changing the real computer.

Example:

Turn.Daemon.exe `
    --test-harness `
    --recipe "Deep Work & Desk Docked" `
    --simulate "ac_disconnect"

The simulated event follows the same logical pipeline:

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

Tests should cover:

- Condition evaluation
- Compound conditions
- Debouncing
- Hysteresis
- Priority conflicts
- State snapshots
- Ownership
- Mode entry
- Mode exit
- Rollback
- Failed actions
- Partial execution
- Device disappearance
- Reordered events
- Duplicate events
- Invalid recipes
- Concurrent modes

---

Fuzz Testing

Recipes and event streams are untrusted inputs and should be fuzz-tested.

Important targets include:

Recipe Parser
    ├── malformed JSON
    ├── unknown condition types
    ├── invalid operators
    ├── deeply nested expressions
    ├── oversized payloads
    └── invalid action parameters

Runtime
    ├── duplicate events
    ├── event storms
    ├── reordered events
    ├── device disappearance
    ├── failed actions
    └── rapid enter/exit cycles

A malformed recipe must never result in arbitrary system modification.

---

Requirements

Windows

- Windows 10 version 2004 / build 19041 or newer
- Windows 11
- Windows SDK 10.0.22621 or newer
- .NET 8 SDK
- Visual Studio 2022 for native Windows components

Android

- Android 8.0 / API 26 or newer
- Bluetooth LE support for BLE transport
- Permissions required by the specific Android features enabled

---

Building

Clone

git clone https://github.com/username/Turn.git
cd Turn

Build the daemon

dotnet build Turn.sln -c Release

For a self-contained Windows build:

dotnet publish `
    src/Turn.Daemon/Turn.Daemon.csproj `
    -c Release `
    -r win-x64 `
    --self-contained true

Build Android

cd src/Turn.Mobile
./gradlew assembleDebug

Install a debug build:

adb install app/build/outputs/apk/debug/app-debug.apk

---

Development

Run the daemon interactively:

Turn.Daemon.exe --run

Run diagnostics:

Turn.Daemon.exe --diagnostics

Run the recipe test harness:

Turn.Daemon.exe `
    --test-harness `
    --recipe "Deep Work & Desk Docked" `
    --simulate "ac_connect"

---

Design Principles

1. Events over polling

If the operating system can tell us that something happened, listen to it.

2. State over commands

A mode represents a desired state, not a collection of irreversible commands.

3. Capture before mutation

Do not assume what the system looked like before Turn changed it.

4. Explicit ownership

Multiple automations must be able to coexist without blindly undoing each other's changes.

5. Local by default

The core automation engine should work without an internet connection.

6. Least privilege

A feature should not require administrator access merely because another feature does.

7. Models do not get root authority

Natural-language generation may create a recipe, but validation and explicit policy remain responsible for execution.

8. Deterministic execution

Given the same initial state, event sequence, and recipe set, the runtime should produce predictable results.

9. Failure is a state

Actions can fail. Devices can disappear. Processes can terminate.

Turn should model those situations rather than assuming every action succeeds.

---

Roadmap

Core Runtime

- [ ] Recipe schema v1
- [ ] Event bus
- [ ] Condition engine
- [ ] Debounce
- [ ] Hysteresis
- [ ] Priority arbitration
- [ ] State ownership
- [ ] Snapshot manager
- [ ] Rollback engine
- [ ] Mock event bus

Windows

- [ ] Power events
- [ ] Network events
- [ ] Display events
- [ ] Audio events
- [ ] USB/device events
- [ ] Process events
- [ ] Foreground-window events
- [ ] Audio enforcer
- [ ] Power-plan enforcer
- [ ] Display enforcer
- [ ] QoS enforcer
- [ ] Workspace enforcer
- [ ] Notification enforcer
- [ ] Network enforcer

Android

- [ ] Companion application
- [ ] Local discovery
- [ ] Device pairing
- [ ] Authenticated transport
- [ ] BLE fallback
- [ ] Battery context
- [ ] Orientation context
- [ ] Proximity context
- [ ] Notification synchronization

UI

- [ ] System tray
- [ ] Recipe editor
- [ ] Active modes
- [ ] Event timeline
- [ ] State inspector
- [ ] Permission manager
- [ ] Device pairing
- [ ] Diagnostics

Local Intelligence

- [ ] Local recipe compiler
- [ ] ONNX inference
- [ ] Recipe schema generation
- [ ] Semantic validation
- [ ] Permission analysis
- [ ] Conflict analysis
- [ ] Natural-language recipe repair

---

Contributing

Contributions are welcome.

When adding a new integration:

1. Keep platform-specific code inside an enforcer or event provider.
2. Prefer native event notifications over polling.
3. Add a mock implementation where practical.
4. Define rollback behavior for reversible state mutations.
5. Handle partial failure explicitly.
6. Validate all recipe input.
7. Add tests for enter, active, exit, and failure states.
8. Document required permissions or privileges.
9. Do not introduce cloud dependencies into the core runtime.
10. Do not add custom cryptographic primitives.

---

License

Turn is open-source software.

The project license will be defined in ""LICENSE"" (LICENSE).

---

Status

«Turn is under active development.»

The architecture described above represents the intended design. Individual Windows and Android integrations may not yet be implemented, and platform restrictions may limit what can be automated without elevated privileges or explicit user permissions.

---

The idea

                    TURN
                     │
                     ▼
             Observe the context
                     │
                     ▼
              Understand the state
                     │
                     ▼
              Enter the right mode
                     │
                     ▼
             Transform the system
                     │
                     ▼
              Context disappears
                     │
                     ▼
              Restore what changed

Turn makes your devices respond to where you are, what you're doing, and what your environment is telling them — without requiring the cloud to be involved.
