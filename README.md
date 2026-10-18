# SBCR Control Dashboard — Setup Notes

Single-file browser dashboard for a **Sequencing Batch Column Reactor** (Feeding →
Aeration → Settling → Withdrawal → Idle) with live sensor gauges, hysteresis
controllers, interlocks, CSV data logging, history charts with phase markers, an
animated tank view, and hardware I/O over the **Web Serial API**.

**Deliverable:** `SBCR-Dashboard.html` — one self-contained file (vanilla JS +
CSS, Chart.js from CDN). No build step, no server required.

---

## 1. Browser requirements

| Requirement | Detail |
|---|---|
| Browser | Chromium-based **Chrome / Edge 89+** (Web Serial). Firefox/Safari: everything works except the hardware link. |
| Secure context | Web Serial requires `https://`, `localhost`, **or opening the file directly (`file://`)** — double-clicking `SBCR-Dashboard.html` works. |
| Linux USB access | Add your user to the `dialout` group: `sudo usermod -aG dialout $USER`, then log out/in. |
| Internet | Only needed for Chart.js (CDN). Without it, gauges/control/logging still work; history plots show a notice. |

## 2. Quick start — simulator (no hardware)

1. Open `SBCR-Dashboard.html` in Chrome/Edge. Mode is **Simulator** by default.
2. Durations are in **seconds**; the time-scale selector (×1 / ×10 / ×60)
   accelerates the whole cycle and the simulated sensor dynamics.
3. Press **Start** → confirm → the batch runs all five phases automatically
   (defaults: 60 + 90 + 90 + 60 + 30 s = 5 min 30 s of process time, i.e. ~33 s
   at ×10).
4. **Repeat** (Cycle configuration panel): **Loop ∞** is ON by default — after
   Idle the sequence restarts at Feeding and keeps running until you press
   **Stop / Pause** (or E-STOP). Uncheck it to stop automatically after a fixed
   number of cycles (1–999). The Sequencer panel shows progress as
   `Cycle k / N` (or `k / ∞`), and "Total remaining" spans all queued cycles.
5. Try the controls: Pause/Resume/Stop/Reset, manual overrides, and the
   **EMERGENCY STOP** (latched; use *Clear E-Stop* to release).

## 3. Connecting real hardware

1. Wire your controller to a USB-serial bridge (CP210x / FTDI / CH340).
2. In the **Serial link** panel: *Choose port…* → select the device → pick the
   baud rate (default 115200) → **Connect**.
3. Switch the mode toggle to **Real** (durations in minutes or hours; time
   scale is fixed at 1×). Mode is locked while a cycle runs.
4. On connect the dashboard sends a full actuator snapshot
   (`HEATER:n`, `SONICATOR:n`, `ACID:n`, `BASE:n`) so firmware and dashboard
   converge. If the link drops, auto-reconnect reopens the port (8 tries,
   2 s apart) and re-sends the snapshot.
5. In real mode the control loop **suspends automatically** (outputs hold)
   if telemetry is missing or stale for > 5 s.

## 4. Serial protocol (line-based, ASCII, LF-terminated)

**device → host** (1–5 Hz recommended):

```
pH:7.21,T:24.5,DO:6.3,S1:0\n
```

* Keys: `pH` (or `ph`), `T` (or `temp`), `DO`, `S1`. Arbitrary key order,
  spacing and unknown keys are tolerated; missing keys keep their last value.

**host → device** (one line per actuator change + snapshot on connect):

```
HEATER:1\n    HEATER:0\n
SONICATOR:1\n SONICATOR:0\n
ACID:1\n      ACID:0\n
BASE:1\n      BASE:0\n
```

### Minimal Arduino firmware example

```cpp
// SBCR firmware example — Arduino Uno/Nano, 115200 baud
// Relays on D5..D8; replace readXXX() with your sensor drivers.
const int PIN_HEATER = 7, PIN_SONIC = 8, PIN_ACID = 6, PIN_BASE = 5;
String buf;
bool heater = 0, sonic = 0, acid = 0, base = 0;

void setup() {
  Serial.begin(115200);
  pinMode(PIN_HEATER, OUTPUT); pinMode(PIN_SONIC, OUTPUT);
  pinMode(PIN_ACID,   OUTPUT); pinMode(PIN_BASE,  OUTPUT);
}

void loop() {
  while (Serial.available()) {                 // ---- command parser ----
    char c = Serial.read();
    if (c == '\n') { handleCmd(buf); buf = ""; } else buf += c;
  }
  static uint32_t t0 = 0;                      // ---- 1 Hz telemetry ----
  if (millis() - t0 >= 1000) {
    t0 = millis();
    float pH = readPH(), T = readTemp(), DO = readDO();  int S1 = readS1();
    Serial.print("pH:");   Serial.print(pH, 2);
    Serial.print(",T:");   Serial.print(T, 1);
    Serial.print(",DO:");  Serial.print(DO, 2);
    Serial.print(",S1:");  Serial.print(S1);
    Serial.print('\n');
  }
}

void handleCmd(String l) {                     // e.g. "HEATER:1"
  l.trim(); l.toUpperCase();
  int i = l.indexOf(':'); if (i < 0) return;
  String k = l.substring(0, i), v = l.substring(i + 1);
  bool on = (v == "1");
  if      (k == "HEATER")    { heater = on; digitalWrite(PIN_HEATER, on); }
  else if (k == "SONICATOR") { sonic  = on; digitalWrite(PIN_SONIC,  on); }
  else if (k == "ACID")      { acid   = on; digitalWrite(PIN_ACID,   on); }
  else if (k == "BASE")      { base   = on; digitalWrite(PIN_BASE,   on); }
}
```

## 5. Control logic & interlocks

| Actuator | Active phase | Logic |
|---|---|---|
| Sonicator | Feeding only | Duty-cycle ON = *SP* s / OFF = *DB* s; force-OFF outside Feeding |
| Heater | all | ON when T < SP−DB · OFF when T > SP+DB (hysteresis) |
| Acid pump | all | ON when pH > SP+DB · OFF when pH < SP−DB |
| Base pump | all | ON when pH < SP−DB · OFF when pH > SP+DB |

* **Acid ✕ base mutual exclusion** — a running pump locks the other out; a
  simultaneous request from OFF blocks both and is reported.
* **Relay guard** — each output toggles at most once per *min-off* seconds
  (configurable, default 3 s of process time). E-stop bypasses it.
* **E-STOP** — forces every output OFF immediately, halts the sequence, and
  latches until cleared.
* Manual overrides force a controller ON (still subject to interlocks/E-stop)
  and are deliberately **not** persisted across reloads.

## 6. Data logging

* Interval selectable (process-time seconds); columns selectable.
* CSV schema (fixed order): `timestamp, phase, pH, temp, DO, S1, heater, acid, base, sonicator`.
* Live preview shows the last 8 rows; buffer holds 8000 rows; **Download CSV**
  exports everything currently buffered.

## 7. Persistence

Durations (both modes), the repeat plan (∞ loop / fixed cycle count), time
scale, gauge ranges/labels/units, all controller setpoints & deadbands, watch
alarms, logging preferences, serial baud and chart window are stored in
`localStorage` (`sbcr.settings.v1`). Reset via DevTools:
`localStorage.removeItem('sbcr.settings.v1')`.

## 8. Safety notice

This dashboard is a research tool. Software E-stop and interlocks do not
replace hardware limits, independent over-temperature cutouts, or properly
rated relay wiring. Never rely on a browser process as the sole safety layer
for a live reactor.
