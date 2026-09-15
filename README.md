# Turn

> Event-driven workspace automation and context-aware state synchronization across Windows and Android.

Turn is an open-source, zero-cloud desktop daemon that brings native **Modes & Routines** to Windows.

Instead of relying on polling loops, manual toggles, or heavyweight scripting environments, Turn listens directly to native hardware and operating-system events. It applies system-level state transformations and seamlessly relays context to an Android companion device over an encrypted, peer-to-peer local connection.

When a trigger fires, Turn records an **atomic snapshot** of the relevant environment. When the condition exits, the previous configuration is restored.

---

## ✨ Highlights

- **Zero-Polling Hardware Bus** — Binds to kernel and Win32 event brokers such as `WM_POWERBROADCAST`, `INetworkEvents`, `WM_DISPLAYCHANGE`, and Core Audio APIs. Designed to remain effectively idle when nothing happens.
- **Deterministic Rollback Stack** — Every mode entry pushes modified properties onto an in-memory stack, allowing previous state to be restored without configuration drift.
- **Peer-to-Peer Cross-Device Sync** — Communicates directly with Android over local Wi-Fi using TLS/TCP, with BLE GATT as a fallback. No cloud account, external broker, or telemetry is required.
- **Declarative Configuration** — Define automation pipelines using human-readable JSON recipes with compound conditions, hysteresis, debounce windows, and priority overrides.
- **Headless Recipe Synthesis** — An optional sub-billion-parameter local model compiler can translate natural-language descriptions into schema-validated recipes entirely offline.
- **Hardware-Aware Automation** — React to displays, power state, audio devices, USB peripherals, network topology, foreground applications, and more.
- **System-Level Enforcement** — Modify audio routing, performance policies, display settings, notifications, workspace state, firewall rules, and other Windows configuration.

---

## 🧠 Philosophy

Turn is built around a simple idea:

> **Your computer already knows when something happened. Turn should listen instead of constantly asking.**

Traditional automation software often relies on:

```text
while true:
    check_condition()
    sleep(...)

Turn instead aims for:

OS / Hardware Event
        ↓
Trigger
        ↓
Condition Evaluation
        ↓
Priority Arbitration
        ↓
State Snapshot
        ↓
Action Enforcement
        ↓
Rollback on Exit

This makes automations event-driven, deterministic, and easier to reason about.


---

🏗️ Architecture

┌───────────────────────────────┐
                      │     SYSTEM & HARDWARE BUS     │
                      │                               │
                      │  ACPI Power                   │
                      │  Network Topology             │
                      │  Audio Sessions               │
                      │  Display / EDID               │
                      │  USB / HID Devices            │
                      │  Process / Workspace          │
                      └───────────────┬───────────────┘
                                      │
                                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│                            TURN CORE DAEMON                            │
│                                                                       │
│  ┌────────────────────┐   ┌────────────────────┐   ┌───────────────┐ │
│  │ Event Dispatcher   │──▶│ Condition Engine   │──▶│ Priority Bus  │ │
│  │                    │   │                    │   │ & Arbiter     │ │
│  │ Win32 / ETW /      │   │ Compound logic     │   │               │ │
│  │ Power / Audio /    │   │ Debounce           │   │ Conflicts     │ │
│  │ Device events      │   │ Hysteresis         │   │ Overrides     │ │
│  └────────────────────┘   └────────────────────┘   └───────┬───────┘ │
│                                                             │         │
│                                                             ▼         │
│                                               ┌─────────────────────┐ │
│                                               │ Snapshot & Rollback │ │
│                                               │ Stack               │ │
│                                               └──────────┬──────────┘ │
└──────────────────────────────────────────────────────────┼────────────┘
                                                           │
                              ┌────────────────────────────┼────────────────────┐
                              │                            │                    │
                              ▼                            ▼                    ▼
                 ┌──────────────────────┐     ┌──────────────────────┐  ┌───────────────┐
                 │ WINDOWS ENFORCERS    │     │ MOBILE SYNC RELAY    │  │ LOCAL PARSER  │
                 │                      │     │                      │  │               │
                 │ Core Audio           │     │ UDP Discovery        │  │ Natural       │
                 │ Display              │     │ TLS/TCP              │  │ Language →    │
                 │ QoS / Affinity       │     │ BLE GATT             │  │ JSON Recipe   │
                 │ Power Plans          │     │ Android DND          │  │               │
                 │ Notifications        │     │ Battery / Proximity  │  │ Offline       │
                 │ Workspace / Shell    │     │ Device Context       │  │ ONNX          │
                 │ Network / WFP        │     │                      │  │               │
                 └──────────────────────┘     └──────────┬───────────┘  └───────────────┘
                                                         │
                                                         │ Encrypted P2P
                                                         ▼
                                                ┌──────────────────────┐
                                                │ ANDROID COMPANION    │
                                                │                      │
                                                │ DND / Notifications  │
                                                │ Battery              │
                                                │ Orientation          │
                                                │ Proximity / BLE      │
                                                │ Hotspot              │
                                                └──────────────────────┘


---

⚡ Event-Driven Core

Turn avoids periodic polling wherever the underlying Windows subsystem exposes an event-driven mechanism.

The core daemon consists of:

Win32 Event Dispatcher

Responsible for receiving native operating-system events through mechanisms including:

SetWinEventHook

RegisterPowerSettingNotification

WM_POWERBROADCAST

WM_DISPLAYCHANGE

WM_DEVICECHANGE

Windows Event Tracing (ETW)

Core Audio notifications

Network event interfaces


Hysteresis & Debounce Controller

Prevents unstable conditions from repeatedly entering and exiting a mode.

For example:

Monitor connected
      ↓
Wait 1500 ms
      ↓
Still connected?
      ├── No  → Ignore
      └── Yes → Enter mode

Priority Bus & Conflict Arbiter

Multiple recipes can react to the same event.

Turn therefore assigns recipes priorities and resolves conflicting actions before enforcement.

Example:

Gaming Mode        priority: 90
Deep Work          priority: 80
Normal Workspace   priority: 10

If Gaming Mode and Deep Work both attempt to modify the same property, the higher-priority state wins.

Snapshot & Rollback Stack

Before modifying state, Turn captures the original value.

Initial State
     │
     ▼
┌─────────────────────┐
│ Volume = 80          │
│ Power = Balanced     │
│ Refresh = 60 Hz      │
│ DND = Off            │
└──────────┬──────────┘
           │
           │ Enter Mode
           ▼
┌─────────────────────┐
│ Volume = 40          │
│ Power = High         │
│ Refresh = 144 Hz     │
│ DND = Priority Only  │
└──────────┬──────────┘
           │
           │ Exit Mode
           ▼
┌─────────────────────┐
│ Restore Snapshot     │
│ Volume = 80          │
│ Power = Balanced     │
│ Refresh = 60 Hz      │
│ DND = Off            │
└─────────────────────┘

This avoids relying on hard-coded "restore" values that may overwrite changes made by the user or another automation.


---

🔌 Trigger Engine

Turn can react to multiple classes of events.

🔋 Power & Battery

AC connection/disconnection

Battery charging state

Battery percentage thresholds

Discharge-rate anomalies

Power-source changes


Primary Windows mechanisms include:

RegisterPowerSettingNotification
WM_POWERBROADCAST


---

🌐 Network & Topology

Supported trigger concepts include:

Wi-Fi SSID changes

BSSID / access-point transitions

Ethernet link changes

Network adapter changes

VPN tunnel interfaces

Network availability


Example:

{
  "type": "network.wifi",
  "operator": "equals",
  "ssid": "Home-5G"
}


---

🖥️ Display & Graphics

Turn can respond to:

External monitor connection

Monitor removal

EDID changes

Specific monitor identification

HDR state

Refresh-rate constraints

Display topology changes


Example:

{
  "type": "hardware.display",
  "operator": "connected",
  "edid": "DEL41A8"
}


---

🎧 Audio & Peripherals

Possible triggers include:

Audio endpoint changes

Audio session creation

Microphone session activation

USB device insertion/removal

Specific USB VID/PID devices


Example:

USB DAC connected
       ↓
Turn detects device
       ↓
Deep Work recipe activated
       ↓
Default audio endpoint switched


---

🪟 Process & Workspace

Turn can react to:

Foreground application changes

Process creation

Process termination

Full-screen applications

Workspace transitions


Windows foreground changes can be observed through:

EVENT_SYSTEM_FOREGROUND


---

📱 Mobile Inbound Events

The Android companion can provide context back to Windows.

Examples:

Incoming calls

Android battery thresholds

Device orientation

Phone proximity

BLE RSSI boundaries

Hotspot state

Device connection/disconnection


This enables automations such as:

Phone placed face-down
        ↓
Android → Turn
        ↓
Activate Focus Mode
        ↓
Windows DND + audio attenuation


---

⚙️ Action Engine

Triggers describe when something happens.

Enforcers describe what Turn does about it.

🔊 Audio

Possible actions include:

Change default multimedia endpoint

Change communications endpoint

Master volume adjustment

Per-application volume ducking

Microphone mute

Endpoint routing


Relevant Windows interfaces include Core Audio APIs and IAudioSessionNotification.


---

🚀 Performance & QoS

Turn can manipulate process behavior using Windows facilities such as:

ProcessPowerThrottling

CPU affinity

Foreground/background QoS

Windows power schemes


Example:

Coding IDE focused
        ↓
Foreground QoS
        ↓
Higher scheduling priority / preferred cores


---

🖥️ Workspace & Shell

Possible actions include:

Virtual desktop transitions

Window positioning

Window snapping

Desktop icon visibility

Dark/light theme changes

Display configuration



---

🔐 Security & Network Boundaries

Turn can optionally perform system-boundary actions such as:

Activate/deactivate local firewall rules

Modify Windows Filtering Platform configuration

Flush DNS cache

Lock workstation

Scrub clipboard contents


These actions should be treated as privileged operations and explicitly authorized by the user.


---

📱 Android Outbound Controls

Turn can relay actions to the Android companion, including:

Enable/disable Do Not Disturb

Mute notification/ringer/media streams

Trigger haptic confirmation

Manage hotspot state

Synchronize workspace state


Android permissions and OS restrictions apply to actions available to third-party applications.


---

📜 Declarative Recipes

Recipes are stored locally at:

%USERPROFILE%\.turn\recipes.json

A recipe describes:

1. What conditions must be satisfied.


2. How the recipe should behave under conflicting conditions.


3. Which actions should execute.


4. Whether each action should participate in rollback.



Example

{
  "$schema": "https://raw.githubusercontent.com/username/Turn/main/schema/recipe.v1.json",
  "name": "Deep Work & Desk Docked",
  "priority": 80,
  "debounce_ms": 1500,

  "conditions": {
    "all": [
      {
        "type": "hardware.display",
        "operator": "connected",
        "edid": "DEL41A8"
      },
      {
        "type": "network.wifi",
        "operator": "equals",
        "ssid": "Home-5G"
      },
      {
        "type": "hardware.power",
        "operator": "equals",
        "source": "ac"
      }
    ]
  },

  "actions": [
    {
      "enforcer": "windows.audio",
      "target": "endpoint",
      "device_id": "{0.0.0.00000000}.{usb_audio_dac_guid}",
      "rollback": true
    },
    {
      "enforcer": "windows.display",
      "refresh_rate_hz": 144,
      "rollback": true
    },
    {
      "enforcer": "windows.notifications",
      "focus_assist": "priority_only",
      "rollback": true
    },
    {
      "enforcer": "android.sync",
      "action": "set_dnd",
      "payload": {
        "enabled": true,
        "suppress_visual": true
      },
      "rollback": true
    }
  ]
}


---

🧩 Recipe Lifecycle

A recipe follows this lifecycle:

┌──────────────────────┐
                    │       INACTIVE       │
                    └──────────┬───────────┘
                               │
                         Event received
                               │
                               ▼
                    ┌──────────────────────┐
                    │ CONDITION EVALUATION │
                    └──────────┬───────────┘
                               │
                        Conditions true?
                         ┌─────┴─────┐
                        No           Yes
                        │             │
                        ▼             ▼
                     Ignore      Debounce /
                                  Hysteresis
                                      │
                                      ▼
                             ┌────────────────┐
                             │ PRIORITY CHECK │
                             └───────┬────────┘
                                     │
                                     ▼
                             Snapshot State
                                     │
                                     ▼
                              Apply Actions
                                     │
                                     ▼
                            ┌────────────────┐
                            │     ACTIVE     │
                            └───────┬────────┘
                                    │
                              Exit condition
                                    │
                                    ▼
                            Restore Snapshot
                                    │
                                    ▼
                             ┌──────────────┐
                             │   INACTIVE   │
                             └──────────────┘


---

📡 Peer-to-Peer Mobile Sync

Turn does not require a cloud service.

The Windows daemon and Android companion communicate directly.

┌───────────────────┐
│   Windows PC      │
│                   │
│   Turn Daemon     │
└─────────┬─────────┘
          │
          │ Local Wi-Fi
          │ UDP Discovery
          │ TLS/TCP
          │
          │ fallback
          │
          │ BLE GATT
          │
          ▼
┌───────────────────┐
│  Android Device   │
│                   │
│  Turn Companion   │
└───────────────────┘

Pairing

The initial pairing process is designed around local verification:

Windows                         Android
   │                               │
   │◄──── Local Discovery ─────────│
   │                               │
   │──── Ephemeral SAS ───────────►│
   │                               │
   │◄──── User Verification ───────│
   │                               │
   │──── Ed25519 Key Exchange ────►│
   │                               │
   │◄════ Encrypted Session ══════►│

After pairing, communication uses authenticated cryptography.

The intended transport stack is:

Wi-Fi
  │
  ├── UDP Discovery
  │
  └── TLS/TCP
          │
          ▼
   Application Protocol

BLE fallback
  │
  └── GATT Characteristic Pipe

Traffic is intended to be protected with:

Ed25519 identity keys

ChaCha20-Poly1305 authenticated encryption

Ephemeral session material

Short Authentication String (SAS) verification


No cloud relay is required.


---

🤖 Optional Local Recipe Compiler

Turn can optionally include a small local model that translates natural-language automation requests into structured recipes.

For example:

User:

"When I connect my monitor at home while plugged in,
put my PC into deep work mode and silence my phone."

             │
             ▼
       Local Model
             │
             ▼
      Recipe Compiler
             │
             ▼
      Schema Validator
             │
             ▼
       Valid Recipe
             │
             ▼
          Turn Core

The model is optional and runs locally.

It should never be trusted as the final authority over system actions.

The compiler pipeline therefore follows:

Natural Language
       ↓
Local Model
       ↓
Structured Recipe
       ↓
JSON Schema Validation
       ↓
Semantic Validation
       ↓
Conflict / Permission Validation
       ↓
User Approval
       ↓
Recipe Installation


---

🗂️ Project Structure

Turn/
├── src/
│   │
│   ├── Turn.Daemon/
│   │   ├── Core/
│   │   │   ├── StateMachine
│   │   │   ├── PriorityArbiter
│   │   │   ├── SnapshotStack
│   │   │   └── RecipeRuntime
│   │   │
│   │   ├── Triggers/
│   │   │   ├── Power
│   │   │   ├── Network
│   │   │   ├── Display
│   │   │   ├── Audio
│   │   │   ├── Devices
│   │   │   ├── Processes
│   │   │   └── Workspace
│   │   │
│   │   ├── Enforcers/
│   │   │   ├── Audio
│   │   │   ├── Display
│   │   │   ├── Power
│   │   │   ├── QoS
│   │   │   ├── Notifications
│   │   │   ├── Workspace
│   │   │   └── Network
│   │   │
│   │   └── Relay/
│   │       ├── Discovery
│   │       ├── TLS
│   │       ├── Protocol
│   │       └── BLE
│   │
│   ├── Turn.Mobile/
│   │   ├── app/
│   │   │   └── src/
│   │   │       └── main/
│   │   │           └── kotlin/
│   │   └── gradle/
│   │
│   ├── Turn.Parser/
│   │   ├── Model
│   │   ├── Compiler
│   │   ├── SchemaValidator
│   │   └── SemanticValidator
│   │
│   └── Turn.UI/
│       ├── Views
│       ├── ViewModels
│       ├── Tray
│       └── Settings
│
├── schema/
│   └── recipe.v1.json
│
├── tests/
│   ├── Core/
│   ├── Triggers/
│   ├── Enforcers/
│   ├── Relay/
│   ├── Recipes/
│   ├── MockHardware/
│   └── Fuzz/
│
├── docs/
├── LICENSE
└── README.md


---

🛠️ Getting Started

Requirements

Windows Host

Windows 10, build 19041 or newer

Windows 11

Visual Studio 2022 with C++ Desktop Development workload or

.NET 8 SDK

Windows SDK 10.0.22621.0 or newer


Android Companion

Android 8.0 / API 26 or newer

BLE peripheral support for BLE fallback



---

📦 Building the Windows Daemon

Clone the repository:

git clone https://github.com/username/Turn.git
cd Turn

Build a self-contained release:

dotnet publish `
    src/Turn.Daemon/Turn.Daemon.csproj `
    -c Release `
    -r win-x64 `
    --self-contained

The resulting executable will be located under:

src/Turn.Daemon/bin/Release/


---

▶️ Running Turn

Initialize the default recipes and install the daemon:

.\src\Turn.Daemon\bin\Release\net8.0-windows\win-x64\publish\Turn.Daemon.exe --install

Run interactively during development:

Turn.Daemon.exe --run

Display diagnostic information:

Turn.Daemon.exe --diagnostics


---

📱 Building the Android Companion

Navigate to the mobile project:

cd src/Turn.Mobile

Build the debug APK:

./gradlew assembleDebug

Install it on a connected device:

adb install app/build/outputs/apk/debug/app-debug.apk


---

🔐 Local Pairing

1. Start Turn on the Windows workstation.


2. Launch the Android companion.


3. Ensure Wi-Fi or Bluetooth is enabled.


4. Select Pair Workstation.


5. Select the discovered Windows workstation.


6. Compare the displayed Short Authentication String (SAS).


7. Approve the pairing on both devices.


8. Turn establishes the local cryptographic identity.


9. Subsequent communication uses the authenticated local connection.



Turn does not require:

A cloud account

A central server

An external message broker

Telemetry

Internet connectivity for normal operation



---

🧪 Verification & Test Harness

Turn includes a hardware-independent simulation environment for testing recipes without changing the real system.

Run the mock hardware assertion suite:

Turn.Daemon.exe `
    --test-harness `
    --recipe "Deep Work & Desk Docked" `
    --simulate "ac_disconnect"

A simulated event should pass through the same logical pipeline as a real event:

Simulated Event
      ↓
Trigger Dispatcher
      ↓
Condition Engine
      ↓
Priority Arbiter
      ↓
Snapshot Manager
      ↓
Mock Enforcer
      ↓
Assertion
      ↓
Rollback Verification

The test harness should verify:

Condition matching

Debounce behavior

Hysteresis

Priority resolution

Snapshot creation

Action ordering

Rollback integrity

Recipe conflicts

Invalid recipes

Repeated enter/exit cycles

Hardware event ordering

Failure recovery



---

🧪 Fuzzing

Recipe parsing and event handling should be fuzz-tested because Turn processes externally influenced state and declarative configuration.

Areas suitable for fuzzing include:

JSON Recipe Parser
        │
        ├── malformed JSON
        ├── deeply nested conditions
        ├── unknown operators
        ├── invalid action payloads
        ├── conflicting priorities
        └── extreme debounce values

Event Dispatcher
        │
        ├── duplicate events
        ├── reordered events
        ├── rapid state changes
        └── unexpected device removal

A malformed recipe should never result in an uncontrolled system mutation.


---

🔄 Rollback Guarantees

Turn's rollback model is based on captured state rather than assumptions.

Instead of:

Enter mode:
    Set volume = 30

Exit mode:
    Set volume = 100

Turn aims to perform:

Enter mode:
    Capture volume = 67
    Set volume = 30

Exit mode:
    Restore volume = 67

This distinction matters when the user's configuration changes before the mode exits.

Nested Modes

Nested state changes can be represented as a stack:

Initial
  │
  ├── Volume: 70
  └── Power: Balanced
        │
        ▼
Deep Work
  │
  ├── Snapshot #1
  ├── Volume: 40
  └── Power: High Performance
        │
        ▼
Presentation Mode
  │
  ├── Snapshot #2
  ├── Volume: 25
  └── Power: Balanced

When Presentation Mode exits:

Restore Snapshot #2
        ↓
Return to Deep Work

When Deep Work exits:

Restore Snapshot #1
        ↓
Return to Initial State

The implementation must additionally detect stale snapshots and external mutations so that rollback does not blindly overwrite legitimate user changes.


---

🧭 Design Principles

1. Event-driven first

If Windows exposes an appropriate event mechanism, Turn should prefer it over polling.

2. Local first

Core automation should continue working without an internet connection.

3. User state is sacred

Turn should restore what the user actually had rather than a hard-coded approximation.

4. Declarative over imperative

Recipes should describe desired behavior rather than embedding arbitrary scripts.

5. Explicit privileges

Actions requiring elevated privileges should be clearly identified and separately authorized.

6. Fail closed

Invalid or ambiguous recipes should not produce uncontrolled system mutations.

7. Deterministic execution

The same event sequence and starting state should produce the same resulting state.

8. Replaceable components

Triggers, enforcers, transports, parsers, and models should be modular rather than deeply coupled.

9. No unnecessary cloud

A local automation daemon should not need a remote service simply to react to a local event.


---

🔒 Security Model

Turn operates close to the operating system and therefore treats security as a first-class concern.

Important boundaries include:

┌─────────────────────┐
                 │ Untrusted Recipe    │
                 └──────────┬──────────┘
                            │
                            ▼
                    Schema Validation
                            │
                            ▼
                   Semantic Validation
                            │
                            ▼
                  Permission Evaluation
                            │
                            ▼
                    Action Dispatcher
                            │
                            ▼
                 Privileged OS Enforcer

The Android connection should similarly treat the phone as an authenticated peer rather than automatically trusted network traffic.

Security-sensitive operations should include:

Authentication

Authorization

Replay protection

Secure key storage

Transport encryption

Pairing verification

Input validation

Least-privilege execution

Explicit permission boundaries

Audit-friendly event logging



---

📊 Resource Goals

Turn is designed to behave like a native operating-system component rather than a continuously active scripting runtime.

Target characteristics:

Component	Goal

Core daemon idle CPU	<0.1% target
Cloud dependency	None
External broker	None
Polling loops	Avoided where native events exist
Configuration	Local JSON
Mobile sync	Local P2P
Recipe compiler	Optional
Model execution	Local/offline
Rollback	In-memory snapshots
Hardware testing	Mockable


These are engineering targets rather than guarantees; actual resource consumption depends on enabled listeners, hardware, Windows configuration, and workload.


---

🧱 Technology Stack

Component	Technology

Windows Daemon	.NET 8 / native Windows APIs
Windows UI	WinUI 3
System Integration	Win32 / Windows SDK
Event Sources	Win32 / ETW / Windows subsystem APIs
Audio	Windows Core Audio
Network	Windows networking APIs
Security	Windows security primitives + modern cryptography
Mobile	Kotlin
Android	API 26+
Mobile Transport	Wi-Fi + BLE GATT
Recipe Format	JSON
Schema	JSON Schema
Local Model	ONNX
Testing	Unit tests + mock hardware + fuzzing



---

🗺️ Roadmap

Phase 1 — Core

[ ] Recipe schema

[ ] Event dispatcher

[ ] Condition engine

[ ] Debounce / hysteresis

[ ] Priority arbiter

[ ] Snapshot manager

[ ] Rollback engine

[ ] Mock event bus


Phase 2 — Windows Integration

[ ] Power triggers

[ ] Network triggers

[ ] Display triggers

[ ] Device triggers

[ ] Audio triggers

[ ] Foreground-process triggers

[ ] Audio enforcer

[ ] Display enforcer

[ ] Power-plan enforcer

[ ] QoS enforcer

[ ] Notification enforcer


Phase 3 — Mobile

[ ] Android companion

[ ] Local discovery

[ ] Secure pairing

[ ] TLS/TCP transport

[ ] BLE fallback

[ ] Battery events

[ ] Orientation events

[ ] Proximity events

[ ] Android DND integration


Phase 4 — UI

[ ] System tray integration

[ ] Recipe editor

[ ] Active-mode dashboard

[ ] Event timeline

[ ] Snapshot inspector

[ ] Permission management

[ ] Pairing UI


Phase 5 — Local Intelligence

[ ] Natural-language recipe generation

[ ] Local ONNX model

[ ] Schema validation

[ ] Semantic validation

[ ] Human approval workflow

[ ] Recipe repair suggestions



---

🤝 Contributing

Contributions are welcome.

Before submitting a pull request:

1. Keep platform-specific code isolated.


2. Add tests for new trigger/enforcer behavior.


3. Avoid introducing polling when an event-driven API exists.


4. Ensure new state mutations have rollback semantics where appropriate.


5. Validate all externally supplied recipe data.


6. Document required Windows permissions.


7. Test failure and device-disconnect scenarios.


8. Keep the daemon functional without cloud services.




---

⚠️ Platform Notes

Turn interacts with operating-system APIs that may have different capabilities or restrictions across Windows versions.

Some functionality may require:

Administrator privileges

Specific Windows APIs

Hardware support

User-granted Android permissions

Android foreground-service permissions

Bluetooth permissions

Windows-specific interfaces that are not officially documented


Features should therefore degrade gracefully when the underlying platform does not expose the required capability.


---

📄 License

Turn is intended to be released as open-source software.

Choose and document the project's license in LICENSE before public release.


---

🌐 Project

Turn
Event-driven automation for Windows, synchronized locally with Android.

Native Events
     ↓
Context
     ↓
Conditions
     ↓
Priority
     ↓
Snapshot
     ↓
Enforcement
     ↓
P2P Synchronization
     ↓
Deterministic Rollback

> Turn listens to your environment, transforms it when the context changes, and puts it back when the moment is over.
