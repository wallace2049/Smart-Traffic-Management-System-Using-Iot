flowchart TD
    P([Power On / Reset]) --> A["setup() — runs once"]
    A --> A1["Serial.begin(115200)<br>print startup banner"]
    A1 --> A2["setupPins()<br>LED pins → OUTPUT<br>IR pins → INPUT_PULLUP<br>all lanes → RED"]
    A2 --> A3["startNormalCycle()<br>Lane 1 → GREEN, others RED<br>Lane 1 count = 0<br>start 5 s GREEN timer"]
    A3 --> A4{"WiFi connected?"}
    A4 -->|"No — retry every 500 ms"| A5["wait, print '.'"]
    A5 --> A4
    A4 -->|"Yes"| A6["print IP address<br>start web server :80"]
    A6 --> L0(["loop() — repeats forever"])

    L0 --> L1["① server.handleClient()<br>serve dashboard to browsers"]
    L1 --> L2["② checkIR()<br>count vehicles on RED lanes"]
    L2 --> L3{"③ Any RED lane<br>with count ≥ 2?"}
    L3 -->|"Yes"| L4["arm preemption:<br>ALL lanes → RED<br>remember the crowded lane<br>start 2 s timer"]
    L3 -->|"No"| L6["⑤ handleNormalCycle()<br>advance normal timings"]
    L4 --> L5{"④ 2 s all-red<br>elapsed?"}
    L5 -->|"No"| L6
    L5 -->|"Yes"| L5a["crowded lane → GREEN<br>others stay RED<br>reset its count<br>resume normal cycle from it"]
    L5a --> L6
    L6 --> L7{"⑥ 1 s since last<br>status print?"}
    L7 -->|"Yes"| L8["print lane states<br>+ vehicle counts"]
    L7 -->|"No"| L9["⑦ delay(10)"]
    L8 --> L9
    L9 --> L0

