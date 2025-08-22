# GW2 Tournament System – Functional & Technical Documentation

## Overview
This document describes the **functionality** and **technical architecture** of a tournament system designed for **Guild Wars 2 racing events**.  
The system consists of two main applications:  
- A **Client Application** (runs on each racer's computer)  
- A **Server Application** (tournament organizer/host), supported by a lightweight **central communication hub** (WebSocket/MQTT).  

The communication hub is **stateless** and only transports messages; the **organizer’s machine** remains the authority for race logic, validation, storage, and replays.

---

## 🏁 Client Application

### Purpose
Runs on each racer's PC to:  
- Read **real-time position** from **GW2 MumbleLink API**  
- Send race data to the tournament server via **WebSocket/MQTT**  
- Display messages, countdowns, and live ranking  

### Startup Operations
1. **NTP Synchronization**  
   - Syncs with NTP server to ensure accurate UTC time  
   - Aligns client clocks with organizer’s reference  

2. **Server Connection**  
   - Prompts user for server code (tournament ID)  
   - Prompts user for mode: **Spectator** or **Racer**  
   - Subscribes to hub topics: `race/{id}/state`, `race/{id}/control`, `race/{id}/ranking`  

3. **Status Monitoring**  
   - Sends ready/heartbeat status (`presence`) every second  
   - Optionally measures ping/latency  
   - Sends ping values every 10 seconds  

### Communication Protocol

#### Incoming Commands (from Server/Organizer)

| Command   | Format                  | Transport           | Description                          |
|-----------|-------------------------|---------------------|--------------------------------------|
| `msg`     | `msg "text"`            | `race/{id}/control` | Displays a message on screen         |
| `start`   | `start t0_timestamp`    | `race/{id}/control` | Receives exact race start timestamp  |
| `ranking` | `ranking json`          | `race/{id}/ranking` | Receives live race ranking           |

#### Outgoing Commands (to Server/Organizer)

- **Position Updates**  
  - Sent every 100ms (QoS0, no retention)  
  - Only Racers send, Spectators do not  
  - Format: `pos x,y,z,speed,utc_timestamp`  
  - Source: MumbleLink position, velocity  

- **Checkpoint Events**  
  - Sent when player crosses checkpoint radius  
  - Format: `checkpoint cp_id,ts,x,y,z,speed`  
  - Verified locally before sending  

- **Presence / Heartbeat**  
  - Sent every 1s (QoS1, retained for LWT)  
  - Format: `{"playerId":"p1","status":"ready"}`  

---

## 🎮 Server Application (Organizer Host)

### Purpose
Runs on the tournament organizer’s PC, provides GUI for tournament setup, validates data, records logs, and generates replays.  

### Startup Operations
1. **Tournament Setup**
   - Window for tournament management  
   - Inputs: Tournament name, Map, Race ID (server code)  

2. **Stage Initialization**
   - Creates 3 tournament stages  
   - Organizer starts at Stage 1  

---

### Tournament Stages

#### Stage 1: Pre-Race 🚦
- **Client Management**  
  - Monitor connected clients & readiness  
- **Communication**  
  - Send announcements via `control` topic  
- **Race Configuration**  
  - Laps (default 1)  
  - Start delay (default 1 min)  
  - Organizer publishes `START` with `t0`  

#### Stage 2: Active Race 🏃‍♂️
- **Real-time Monitoring**  
  - Interactive map with all positions (from `pos`)  
  - Live ranking with checkpoint/timers (from `checkpoint`)  
- **Checkpoint System**  
  - Organizer validates order/timestamps  
  - Discards impossible speeds/teleports  

#### Stage 3: Post-Race 🏆
- **Results & Analysis**  
  - Final ranking (stored locally as `.jsonl.gz`)  
  - Race replay: playback positions from recorded stream  
  - Heatmaps, ghost replays, per-checkpoint stats  

---

## 🔄 System Flow

```
Client Startup
→ Hub Connection (MQTT/WebSocket)
→ Ready Status
→ Organizer publishes START (t0)
→ Position Updates / Checkpoint Events
→ Organizer validates & records
→ Final Ranking + Replay
```


---

## ⚙️ Communication Hub

- **Role:** Relay only (no heavy logic)  
- **Options:**  
  - MQTT broker (AWS IoT, EMQX Cloud, HiveMQ Cloud)  
  - WebSocket API Gateway (AWS, or similar)  

### Topics Example (MQTT)

```
race/{id}/control # msgs, start, pause
race/{id}/presence # join/leave, LWT
race/{id}/pos/{pid} # position stream
race/{id}/checkpoint # checkpoint events
race/{id}/ranking # live ranking updates
race/{id}/state # retained: map, t0, config
```

### QoS Levels
- `pos`: QoS0 (fast, drop tolerated)  
- `checkpoint`, `control`: QoS1  
- `state`: Retained  

---

## 📋 Key Features

- **Real-time sync** via NTP + server clock offset  
- **MumbleLink** for legal, real-time GW2 telemetry  
- **Live position tracking** (100ms updates, QoS0)  
- **Checkpoint-based ranking** with local + host validation  
- **Replay system**: organizer stores all events as JSONL+gzip  
- **Lightweight hub**: only message transport, no heavy logic  
- **Scalable**: cloud MQTT/WebSocket can handle many clients  
- **Secure**: TLS (mqtts/wss), ephemeral race keys per tournament  

---

## ✅ Advantages of this Model

- **Organizer has full authority** → can generate replays, anti-cheat, results without relying on cloud.  
- **Hub is stateless** → cheap, scalable, replaceable.  
- **Clients stay simple** → only gather MumbleLink data + send.  
- **Open-source friendly** → client (Python/Blish HUD), server (Python/C#/Electron), hub (standard MQTT/WebSocket).  

