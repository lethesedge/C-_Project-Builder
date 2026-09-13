# Hardware Track — ESP32 / ARES (v0.1 sketch)

## Why this track exists

The software tracks (Foundations, Data Structures & Memory, ...) teach C++
in the abstract. This track exists so the same learner can point that C++
at something physical: an ESP32-based rover, sized to be the actual
brainstem of ARES rather than a disconnected toy exercise.

**Prerequisite**: Foundations projects 01-05 (variables, control flow,
functions, arrays, structs) — pointers/classes from 07-09 are *not* required
to start; a couple of early hardware projects (H1-H2) work fine with just
that. Can run in parallel with the back half of Foundations rather than
strictly after it.

**Board target**: ESP32 (any dev board, e.g. ESP32-WROOM DevKit). Chosen
because it's cheap, has built-in WiFi/BT (needed for H6 and the ARES bridge
anyway), and is well supported by the Arduino framework for C++.

## Architecture decision: two brains, not one

ARES's eventual scope (drive, sense, avoid obstacles, take remote commands,
*and* run CV/ML for autonomy) is too much for one chip to do well. The
standard split, and the one this track builds toward:

```
┌─────────────────────────┐        serial or WiFi        ┌──────────────────────────┐
│   ESP32 — "brainstem"    │◀────────────────────────────▶│  Pi/laptop — "brain"     │
│                          │   simple command protocol    │                          │
│  - motor control (PWM)   │   e.g. "DRIVE 80 80\n"       │  - camera + CV/ML         │
│  - ultrasonic/IR sensors │   "STOP\n"                   │  - path planning          │
│  - safety cutoffs        │   "TELEM dist=42 batt=7.4\n" │  - talks to ESP32 over a  │
│  - real-time, no OS-     │                              │    tiny protocol, not     │
│    building needed       │                              │    "runs on it"           │
│    (Arduino/FreeRTOS)    │                              │                          │
└─────────────────────────┘                               └──────────────────────────┘
```

Rationale for *this specific split*:
- The ESP32 must react to "wall in 5cm" in milliseconds regardless of what
  the vision stack is doing — that has to be local, simple, and never
  blocked on a network call or a slow ML inference.
- CV/ML belongs on hardware that can actually run it (a Pi's CPU/GPU, or a
  laptop during early development) — trying to run it on the ESP32 would
  mean fighting the chip instead of learning CV/ML.
- This is exactly the architecture real rovers use (a real-time
  microcontroller layer + a higher-level compute layer), so it's not a
  simplification for teaching purposes — it's the right answer anyway.

A future **Vision & Autonomy** track (not sketched here) would live on the
"brain" side and consume the protocol project H7 establishes. It doesn't
need to be C++ necessarily — that's an open question (§ below).

## Project sequence

| # | Project | Introduces | Builds on |
|---|---------|------------|-----------|
| H1 | Blink & Button, Properly | digital I/O, debouncing, non-blocking timing (`millis()` not `delay()`) | Foundations 01-03 |
| H2 | Motor Driver | PWM (`ledcWrite`), H-bridge control, a `drive(leftSpeed, rightSpeed)` API | H1 |
| H3 | Ultrasonic Distance Sensor | pulse timing, unit conversion, noise filtering (moving average) | Foundations 05 (arrays, for the filter buffer) |
| H4 | **Capstone A**: Obstacle-Avoiding Rover | combining H1-H3 into one non-blocking control loop | H1-H3 |
| H5 | Multitasking Without Rolling Your Own OS | FreeRTOS tasks (`xTaskCreate`), queues for passing sensor data between tasks safely | H4 |
| H6 | Wireless Telemetry & Control | ESP32 WiFi (AP or station mode), a minimal web server or WebSocket streaming sensor data and accepting drive commands | H2, H5 |
| H7 | **Capstone B**: Brainstem/Brain Bridge | defining and implementing the ESP32-side of the serial/WiFi command protocol above; deliverable is the protocol working against a *stub* brain that just sends hardcoded commands | H4-H6 |

### H1 — Blink & Button, Properly
- **Pitch**: blink an LED and read a button, but the *point* is doing it
  without `delay()` — track elapsed time with `millis()` so the loop stays
  responsive. This habit is non-negotiable for everything after it.
- **Concepts**: `pinMode`/`digitalWrite`/`digitalRead`, debouncing a noisy
  button signal, non-blocking timing patterns.
- **Done when**: LED blinks at a fixed interval, button press is correctly
  detected exactly once per physical press (no double-triggers from bounce),
  and there is no `delay()` anywhere in the sketch.

### H2 — Motor Driver
- **Pitch**: spin two DC motors (via an H-bridge driver, e.g. L298N/DRV8833)
  forward, backward, and at variable speed, from a clean `drive(int left,
  int right)` function.
- **Concepts**: PWM via `ledcWrite`, motor driver wiring (direction pins +
  enable/PWM pin), mapping a speed range to duty cycle.
- **Done when**: both motors independently go forward/back/stop at
  controllable speed, and all motor control goes through one function (not
  scattered `digitalWrite` calls) so H4 can call it cleanly.

### H3 — Ultrasonic Distance Sensor
- **Pitch**: read an HC-SR04 (or similar) and print a stable distance
  reading, not a jumpy raw one.
- **Concepts**: `pulseIn` timing, converting pulse width to distance,
  smoothing noisy sensor data with a small moving-average buffer (a real
  array from Foundations 05, put to use).
- **Done when**: readings are accurate to a reasonable tolerance at a few
  known distances, and a single spurious reading doesn't cause a visible
  jump in the smoothed output.

### H4 — Capstone A: Obstacle-Avoiding Rover
- **Pitch**: the rover drives forward and stops/turns before hitting
  something, continuously, using H1-H3 together.
- **Concepts**: none new — integration is the point. One `loop()` that
  polls the sensor, decides on a motor command, and calls `drive()`,
  entirely non-blocking.
- **Done when**: rover reliably avoids a wall/obstacle at a set distance,
  keeps responding to a button (from H1) as an emergency stop even while
  driving, and the loop never uses `delay()`.

### H5 — Multitasking Without Rolling Your Own OS
- **Pitch**: split sensor polling, motor control, and (later) WiFi handling
  into separate FreeRTOS tasks instead of one big `loop()`, communicating
  via a queue.
- **Concepts**: `xTaskCreate`, `FreeRTOS` queues for passing a sensor
  reading from one task to another safely, why this beats "just add more
  logic to loop()" once there are several time-sensitive things to juggle
  at once. This is the answer to "should I build an OS": no — this *is*
  the OS, already there, and worth learning its primitives directly.
- **Done when**: sensor polling and motor control run as separate tasks,
  data crosses between them only via a queue (not a shared global written
  from both sides), and the emergency-stop button still works instantly
  regardless of what the other tasks are doing.

### H6 — Wireless Telemetry & Control
- **Pitch**: connect to WiFi, serve a minimal page or WebSocket that shows
  live sensor readings and accepts drive commands from a phone/laptop.
- **Concepts**: ESP32 WiFi modes, a basic web server or WebSocket loop,
  and — the actual lesson — folding a third concern (networking) into the
  H5 task structure without it blocking motor control.
- **Done when**: sensor data is visible remotely in near-real-time, and a
  remote drive command is reflected within a short, bounded latency.

### H7 — Capstone B: Brainstem/Brain Bridge
- **Pitch**: implement the ESP32 side of a tiny text command protocol
  (`DRIVE l r`, `STOP`, `TELEM ...`) over serial or WiFi, and prove it
  against a stub "brain" (a Python or C++ script on a laptop that just
  sends a scripted sequence of commands — no CV/ML yet).
- **Concepts**: designing a minimal, parseable line protocol; treating the
  ESP32 as a server for commands rather than the decision-maker.
- **Done when**: the stub brain can drive the rover through a full
  forward/turn/stop sequence and receive telemetry back, entirely through
  the protocol (no other coupling between the two sides).
- **This is the seam**: whatever the future Vision & Autonomy track builds
  on the "brain" side plugs in here without touching the ESP32 code again.

## Open questions specific to this track

- **What board/driver exactly?** Sketch assumes a generic ESP32 DevKit +
  L298N/DRV8833 + HC-SR04 since they're the cheapest, best-documented combo
  — worth confirming against whatever ARES's BOM actually is before buying
  anything, since pin counts/voltage levels differ across drivers.
- **Language on the "brain" side (H7 onward).** Python is the fastest path
  to a working CV/ML stub (OpenCV, PyTorch/TFLite bindings are mature) but
  it breaks the "C++ tutor" framing once we cross that seam. Leaning toward:
  let the brain-side prototype be Python (fastest iteration for CV/ML,
  which is a different discipline from systems C++), and treat "port the
  hot path to C++" as an optional, later, systems-track exercise once
  something is working — not a blocker for ARES making progress.
- **Simulation fallback.** If a physical board/motor/sensor isn't in hand
  yet for a given project, is a simulator (e.g. Wokwi, which supports
  ESP32 + these exact sensors) an acceptable substitute so the track isn't
  blocked on hardware shipping? Recommend yes, with a note in each project
  that real hardware is expected eventually before calling it "done" for
  ARES purposes specifically (a simulated obstacle-avoider proves the code;
  ARES needs it proven on the actual rover).
- **Where this track sits relative to the software tracks.** Sketch above
  says "parallel with late Foundations," but H5 (FreeRTOS/concurrency) is
  arguably systems-track material (track 3 in the main design doc). Open
  question whether H5-H7 should be gated behind track 2 instead of running
  right after H4.
