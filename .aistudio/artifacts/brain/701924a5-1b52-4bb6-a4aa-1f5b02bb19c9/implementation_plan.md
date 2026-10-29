# Pixel Sandbox: Virtual Device & Entropy Simulator

An interactive Android developer simulation sandbox modeling a Google Pixel 2 XL environment, featuring live kernel entropy pool telemetry, interactive entropy harvesting, an active process manager with POSIX kill signals, and system stress event simulation.

## User Review & Critical Decisions

> [!IMPORTANT]
> The following parameters were confirmed during interactive clarification and guide the architecture below:

- **Confirmed Concept**: Virtual Android device & entropy sandbox simulation modeling the Google Pixel 2 XL ("taimen").
- **Confirmed Core Scope**: Virtual device sandbox with simulated system events, real-time entropy pool telemetry, and interactive process management with kill signals.
- **Confirmed Visual Direction**: Clean developer dashboard featuring high-density data cards, monospace telemetry values, dynamic status chips, and edge-to-edge Material 3 ergonomics.

---

## 1. Overview & Core Concept

### What It Does
**Pixel Sandbox** is a specialized system simulation and developer playground. It simulates the core architecture and kernel behaviors of a Google Pixel 2 XL running Android:
1. **Google Pixel 2 XL Profile**: Emulates device specifications (Qualcomm Snapdragon 835 MSM8998, 8 cores, Adreno 540, 4 GB LPDDR4X RAM, 6.0" P-OLED 2880×1440 profile), boot states, and system uptime.
2. **Default Source Entropy Sandbox**: Visualizes the Linux kernel random entropy pool (4096-bit capacity), tracking entropy availability (`/proc/sys/kernel/random/entropy_avail`), read rates from `/dev/urandom` vs. `/dev/random`, and source contributions (hardware RNG, interrupt jitter, thermal noise, touch screen motion). Includes an interactive touch/accelerometer entropy harvester and cryptographic randomness tests (monobit frequency, chi-square).
3. **Process Supervisor & Kill Engine**: Simulates authentic Android system and user processes (`init`, `zygote64`, `system_server`, `surfaceflinger`, `logd`, `vold`, `com.google.android.gms`, etc.) with PID, CPU %, RSS memory, and state. Users can dispatch POSIX signals (`SIGKILL -9`, `SIGTERM -15`, `SIGSTOP -19`, `SIGCONT -18`) or invoke mass kill actions, observing how the simulated OS handles service failures (e.g., killing `surfaceflinger` or `system_server` triggers an automated watchdog restart or simulated soft-reboot).
4. **System Event & Stress Simulator**: Enables users to inject synthetic system stress events—such as Low Memory Killer (LMK) triggers, kernel entropy starvation, thermal throttling, and sensor interrupt storms—and watch the device state and live Logcat stream react dynamically.

### Target Audience & Persona
Android developers, security enthusiasts, systems engineers, and students who want an intuitive, tactile tool to understand Android kernel internals, entropy generation, process lifecycle behaviors, and system event handling.

---

## 2. User Experience & Visual Design

### Key User Flows
- **Flow 1: Device Telemetry & Logcat Stream**: View Pixel 2 XL hardware health, active CPU clock frequencies across cluster cores, memory allocation (free, cached, used), and filterable real-time Logcat stream.
- **Flow 2: Entropy Harvesting & Randomness Analysis**: Inspect the live 4096-bit entropy pool. Use the interactive touch jitter pad to manually feed high-entropy seed data into the pool, deplete entropy via simulated `/dev/random` consumers, and execute monobit cryptographic uniformity checks.
- **Flow 3: Process Inspection & Signal Dispatch (Kill)**: Browse the active process table. Tap any process to open the signal modal, select `SIGKILL` or `SIGTERM`, confirm execution, and watch the process state transition, memory deallocation, and system watchdog restart reactions.
- **Flow 4: System Event Injection**: Trigger predefined scenario cards (e.g., "LMK Memory Pressure", "Entropy Pool Starvation", "Sensor Flood"), observing system-wide metrics, alerts, and log events.

### Visual Identity & Theme
- **Aesthetic Direction**: High-density developer dashboard with technical instrumentation, telemetry badges, and monospace data readouts.
- **Color Palette**:
  - Dark Slate Background (`#0F141C`) and Card Surfaces (`#161E2E`, `#1E293B`).
  - Terminal Cyan / Emerald Accents (`#10B981`, `#06B6D4`) for nominal states and entropy flow.
  - Amber / Warning Accents (`#F59E0B`) for memory pressure and CPU spikes.
  - Crimson / Kill Red (`#EF4444`) for kill signals, process termination, and critical errors.
  - High-contrast text tokens compliant with Material Design 3.
- **Typography & Hierarchy**:
  - Monospace formatting (`FontFamily.Monospace`) for PIDs, addresses, bit counts, byte sizes, and timestamps.
  - Clean M3 sans-serif for section titles, descriptions, and interactive button controls.
- **Layout & Density**:
  - High-density cards with minimal padding waste, responsive to compact phones and tablet/wide layouts.
  - Animated level bars, smooth progress rings for entropy fullness, and sparkline charts for pool history.

---

## 3. Key Product Decisions & Trade-Offs

- **Decision 1: Deterministic & Organic Simulation Engine**
  - *Approach*: Built on a dedicated Kotlin `SimulationEngine` running lightweight coroutine ticks (500ms intervals) that update entropy levels, process CPU/memory fluctuations, and event logs smoothly without burdening the host device.
  - *Why*: Provides immediate, hyper-responsive feedback, allowing users to stress-test scenarios safely without crashing the actual Android host environment.
- **Decision 2: Realistic Android Process Hierarchy & Failure Cascades**
  - *Approach*: Core system daemons have defined parent-child dependencies and restart policies. Terminating `init` triggers a virtual kernel panic; terminating `surfaceflinger` triggers a graphical restart logcat; terminating background apps simply reclaims memory.
  - *Why*: Delivers an educational and deeply authentic simulation experience of Android's process architecture.
- **Decision 3: Dual Entropy Sources (Real Device Telemetry + Simulated Kernel Pool)**
  - *Approach*: Combines actual device sensor jitter (gyroscope/accelerometer and touch timestamps) as real seed inputs into the simulated 4096-bit Linux kernel entropy pool model.
  - *Why*: Fulfills the user's explicit request for default source entropy and simulation.

---

## 4. Technical Architecture & Data Strategy

### Architecture & Component Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                          Jetpack Compose UI                            │
├───────────────────┬────────────────────┬─────────────────┬─────────────┤
│ Device Sandbox    │   Entropy Pool     │  Process Table  │ System Event│
│   (Pixel 2 XL)    │ (Harvester/Tests)  │ (Kill Dispatch) │  Scenarios  │
└─────────┬─────────┴──────────┬─────────┴────────┬────────┴───────┬─────┘
          │                    │                  │                │
          ▼                    ▼                  ▼                ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         PixelSandboxViewModel                          │
│   - StateFlow<SandboxUiState>                                          │
│   - Event Dispatchers (killProcess, harvestEntropy, triggerScenario)   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                          Simulation Engine                             │
├────────────────────┬────────────────────┬──────────────────────────────┤
│  DeviceModelManager│  EntropyPoolEngine │     ProcessSupervisor        │
│  - Pixel 2 XL Specs│  - 4096-bit Pool   │     - Process Hierarchy      │
│  - CPU/RAM Telemetry│  - Source Mixers   │     - POSIX Signal Handler   │
│  - Logcat Buffer   │  - Randomness Tests│     - Watchdog & Restart     │
└────────────────────┴────────────────────┴──────────────────────────────┘
```

### Data Models
- `DeviceSpecs`: Hardware identification (Google Pixel 2 XL, taimen, Snapdragon 835 8-core, 4GB RAM, P-OLED display).
- `EntropyPool`: Current available entropy (0–4096 bits), write rate (bits/s), read rate (bits/s), pool depletion state, source contributions (`hw_random`, `interrupts`, `touch_jitter`, `sensor_noise`).
- `SimulatedProcess`: PID, name, user/group, state (`RUNNING`, `SLEEPING`, `STOPPED`, `ZOMBIE`), CPU percentage, RSS memory (MB), OOM score, whether it is a critical system service.
- `SystemScenario`: Pre-configured stress triggers (OOM Spikes, Entropy Starvation, Thermal Limit, Interrupt Flood).
- `LogcatEntry`: Timestamp, log level (`V`, `D`, `I`, `W`, `E`, `F`), tag, message.
