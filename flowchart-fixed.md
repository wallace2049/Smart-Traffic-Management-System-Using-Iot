# Smart Traffic Management System — Program Flowcharts

Flowcharts for the ESP32-based adaptive traffic light controller.

## How the system works

* **Normal cycle** — each lane takes a turn: 5 s GREEN → 5 s YELLOW → RED while the next lane goes. Order: Lane 1 → 2 → 3 → 4 → 1 …
* **Vehicle counting** — while a lane is RED, its IR sensor counts vehicles passing the stop line (one count per vehicle, debounced over 80 ms).
* **Preemption** — if any RED lane accumulates ≥ 2 vehicles, the whole intersection goes ALL-RED for 2 s, then that lane gets GREEN immediately. Afterwards the normal cycle resumes from that lane.
* **Web dashboard** — the ESP32 serves a live status page on port 80 (auto-refreshes every 1 s).

Each `loop()` pass (≈ 10 ms) runs, in order:
`server.handleClient()` → `checkIR()` → `checkPreemption()` → `handlePreemption()` → `handleNormalCycle()` → periodic Serial status print.

## Contents

1. [Main program flow](#1-main-program-flow)
2. [checkIR() vehicle counting](#2-checkir-vehicle-counting)
3. [Preemption path](#3-preemption-path)
4. [handleNormalCycle() state transitions](#4-handlenormalcycle-state-transitions)
5. [Lifecycle of one lane signal](#5-lifecycle-of-one-lane-signal)

---

## 1. Main program flow

```mermaid
flowchart TD
    P(["Power On / Reset"]) --> A["setup() — runs once"]
    A --> A1["Serial.begin(115200)<br/>print startup banner"]
    A1 --> A2["setupPins()<br/>LED pins → OUTPUT<br/>IR pins → INPUT_PULLUP<br/>all lanes → RED"]
    A2 --> A3["startNormalCycle()<br/>Lane 1 → GREEN, others RED<br/>Lane 1 count = 0<br/>start 5 s GREEN timer"]
    A3 --> A4{"WiFi connected?"}
    A4 -->|"No — retry every 500 ms"| A5["wait, print '.'"]
    A5 --> A4
    A4 -->|"Yes"| A6["print IP address<br/>start web server on port 80"]
    A6 --> L0(["loop() — repeats forever"])

    L0 --> L1["① server.handleClient()<br/>serve dashboard to browsers"]
    L1 --> L2["② checkIR()<br/>count vehicles on RED lanes"]
    L2 --> L3{"③ Any RED lane<br/>with count ≥ 2?"}
    L3 -->|"Yes"| L4["arm preemption:<br/>ALL lanes → RED<br/>remember the crowded lane<br/>start 2 s timer"]
    L3 -->|"No"| L6["⑤ handleNormalCycle()<br/>advance normal timings"]
    L4 --> L5{"④ 2 s all-red<br/>elapsed?"}
    L5 -->|"No"| L6
    L5 -->|"Yes"| L5a["crowded lane → GREEN<br/>others stay RED<br/>reset its count<br/>resume normal cycle from it"]
    L5a --> L6
    L6 --> L7{"⑥ 1 s since last<br/>status print?"}
    L7 -->|"Yes"| L8["print lane states<br/>+ vehicle counts"]
    L7 -->|"No"| L9["⑦ delay(10)"]
    L8 --> L9
    L9 --> L0
```

## 2. checkIR() vehicle counting

```mermaid
flowchart TD
    C0(["checkIR()"])
    C0 --> C1["for each lane i = 1 … 4"]
    C1 --> C2{"lane is RED?"}
    C2 -->|"No — counting only happens while RED"| C10["next lane"]
    C2 -->|"Yes"| C3["reading = digitalRead(IR pin)"]
    C3 --> C4{"reading changed<br/>since last check?"}
    C4 -->|"Yes"| C5["record timestamp of change<br/>(restarts debounce window)"]
    C4 -->|"No"| C6{"stable for<br/>more than 80 ms?"}
    C5 --> C6
    C6 -->|"No"| C10
    C6 -->|"Yes"| C7{"reading LOW and<br/>not yet counted?"}
    C7 -->|"Yes — beam just broken"| C8["vehicle count++<br/>irTriggered = true<br/>print COUNT message"]
    C7 -->|"No"| C9{"reading HIGH and<br/>previously counted?"}
    C9 -->|"Yes — beam clear again"| C9a["irTriggered = false<br/>ready for next car"]
    C9 -->|"No"| C10
    C8 --> C10
    C9a --> C10
    C10 --> C11{"more lanes to check?"}
    C11 -->|"Yes"| C2
    C11 -->|"No"| C12(["return"])
```

## 3. Preemption path

```mermaid
flowchart TD
    P0(["checkPreemption()"])
    P0 --> P1{"already in<br/>preemption?"}
    P1 -->|"Yes"| P2(["return — let it finish"])
    P1 -->|"No"| P3{"any RED lane<br/>with count ≥ 2?"}
    P3 -->|"No"| P2
    P3 -->|"Yes — lane L found"| P4["allRed()<br/>even the currently GREEN lane goes RED"]
    P4 --> P5["preemptedLane = L<br/>start 2 s ALL-RED timer"]
    P5 --> P6(["return — wait for the 2 s window"])

    P6 --> Q0(["handlePreemption()<br/>on a later loop pass"])
    Q0 --> Q1{"2 s elapsed?"}
    Q1 -->|"No"| Q2(["keep all lanes RED"])
    Q1 -->|"Yes"| Q3["lane L → GREEN<br/>others stay RED"]
    Q3 --> Q4["reset count of lane L to 0<br/>print RESET message"]
    Q4 --> Q5["currentGreenIndex = L<br/>start 5 s GREEN timer<br/>clear preemption flags"]
    Q5 --> Q6(["normal cycle continues<br/>from lane L"])
```

## 4. handleNormalCycle() state transitions

```mermaid
flowchart TD
    N0(["handleNormalCycle()"])
    N0 --> N1{"preemption<br/>in progress?"}
    N1 -->|"Yes"| N2(["return — preemption owns the lights"])
    N1 -->|"No"| N3{"current phase<br/>timer finished?"}
    N3 -->|"No"| N4(["return — keep current lights"])
    N3 -->|"Yes"| N5{"current lane<br/>state?"}
    N5 -->|"GREEN"| N6["current lane → YELLOW<br/>start 5 s YELLOW timer"]
    N5 -->|"YELLOW"| N7["current lane → RED<br/>advance to next lane<br/>next lane → GREEN"]
    N7 --> N8["reset next lane count to 0<br/>start 5 s GREEN timer"]
    N6 --> N9(["return"])
    N8 --> N9
```

## 5. Lifecycle of one lane signal

```mermaid
stateDiagram-v2
    direction LR
    RED --> GREEN : normal turn arrives OR count ≥ 2 → priority green
    GREEN --> YELLOW : 5 s elapsed
    YELLOW --> RED : 5 s elapsed → next lane
    note right of RED
        While RED: IR sensor counts cars.
        Count resets to 0 the moment the lane turns GREEN.
    end note
```
