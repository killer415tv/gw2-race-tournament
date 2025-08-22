# Racing Telemetry System - Documentation

## Overview
This project consists of two main components:
- A **central application** for organizing and managing racing championships.
- A **lightweight client application** that collects telemetry data from participants during races.

---

## Central Application

### Main Features
- Create and manage tournaments.
- Select the race map and tournament code.
- Send global or individual messages to clients.
- Trigger synchronized race start.
- Monitor checkpoints and determine rankings in real time.
- Show a map with participant positions 2D / 3D

### Responsibilities
- Store checkpoints for each race map.
- Validate when a player reaches a checkpoint.
- Calculate each player's ranking dynamically.
- Control the entire race logic from the server side.
- Protection against multiple instances with the same code. (no more central application can use the same code)

---

## Client Application

### Description
- Designed to be compiled as a simple `.exe` executable.
- Requires no installation.
- User only needs to input a tournament code to start.

### Behavior
- Sends real-time telemetry data (X, Y, Z coordinates, speed, angle, name) to the central server.
- Displays:
  - Countdown to race start.
  - Current ranking whenever passing a checkpoint.
  - Finish results of race, when the race is finished. 
  - Own participation Status. 
- Only shows basic overlays on screen to avoid obstructing the gameplay.

### Technical Details
- Does **not** use Tkinter (to remain lightweight).
- Uses a minimal library (e.g., `pygame` or alternatives) for text display.
- Sends data to the central server using a protocol like **ZeroMQ** or **MQTT**.
- Includes timestamps in telemetry messages for synchronization.

---

## Communication & Synchronization

- All clients must synchronize clocks using NTP or use the server timestamp for reference.
- The server sends a scheduled "race start" message with a fixed timestamp.
- Each telemetry message includes a timestamp for precise tracking and analysis.

---

## File Structure (Example)

```
project/
│
├── central_app/
│ └── main.py
│ └── maps/
│ └── map1_checkpoints.json
│
├── client_app/
│ └── client.py
│ └── display_overlay.py
│
└── shared/
└── config/
└── settings.json`
```


---

## Future Improvements

- Add replay system using timestamped telemetry data.
- Enhance visualization with 3D track rendering.
- Add live spectator view in the central app.
