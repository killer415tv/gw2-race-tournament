# GW2 Tournament System - Functional & Technical Documentation

## Overview
This document describes the **functionality** and **technical architecture** of a tournament system designed for **Guild Wars 2 racing events**.  
The system consists of two main applications:  
- A **Client Application** (runs on each racer's computer)  
- A **Server Application** (tournament organizer/host), supported by a lightweight **central communication hub** (WebSocket/MQTT).  

The communication hub is **stateless** and only transports messages; the **organizer's machine** remains the authority for race logic, validation, storage, and replays.

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
   - Aligns client clocks with organizer's reference  

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
Runs on the tournament organizer's PC, provides GUI for tournament setup, validates data, records logs, and generates replays.  

### Startup Operations
1. **Tournament Setup**
   - Window for tournament management  
   - Inputs: Tournament name, Map, Race ID (server code)  

2. **Stage Initialization**
   - Creates 3 tournament stages  
   - Organizer starts at Stage 1  

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

## 💾 Local Data Storage & Replay System

### Local Storage Architecture
The server application stores **all tournament information locally** on the organizer's machine:

- **Race Traces**: Complete position data from all racers throughout the entire race
- **Race Information**: Tournament metadata, checkpoints, timestamps, participant data
- **Event Logs**: All checkpoint events, start/stop commands, and race state changes
- **Replay Files**: Compressed race recordings for post-race analysis

### Data Storage Format
- **Primary Format**: JSONL (JSON Lines) compressed with gzip/zstd
- **File Naming**: `race_{raceId}_{timestamp}.replay.jsonl.gz`
- **Structure**: One JSON object per line representing a single event or position update

### Replay System Features
The post-race replay system provides a **video-like playback experience**:

#### **Interactive Map Visualization**
- **Real-time Position Playback**: View all racer positions as they move through the race
- **Checkpoint Markers**: Visual representation of all checkpoints on the map
- **Race Track Overlay**: Complete race track visualization with start/finish lines

#### **Time Control Interface**
- **Playback Controls**: Play, pause, fast-forward, rewind, and stop functionality
- **Timeline Slider**: Drag to jump to any specific moment in the race
- **Speed Control**: Adjust playback speed (0.25x, 0.5x, 1x, 2x, 4x)
- **Time Display**: Current race time and total race duration

#### **Advanced Replay Features**
- **Ghost Replays**: Overlay multiple racers' paths simultaneously
- **Heatmaps**: Visualize popular racing lines and checkpoint approaches
- **Sector Analysis**: Break down race into segments for detailed performance review
- **Comparison Mode**: Side-by-side comparison of different racers or race attempts

#### **Data Export & Analysis**
- **Performance Metrics**: Detailed statistics for each racer and checkpoint
- **Export Formats**: CSV, JSON, or custom formats for external analysis tools
- **Screenshot Capture**: Save specific moments or race states as images
- **Video Export**: Generate video files of replays for sharing or analysis

---

## 🏗️ Technical Architecture (Proposed)

### Overview
This section describes the proposed technical architecture for the tournament system, implementing a serverless approach with local authority management.

---

## 🚀 Proposed Architecture (Serverless + Local Authority)

### Client Applications (Players)
- **Lightweight overlay**: Blish HUD module in C# or Python app reading MumbleLink for position/rotation/map data
- **Organizer mode**: Same app but with "host" role: defines track/checkpoints, marks start line, and records entire stream (positions/events) locally for replay and post-tournament analysis

### Central Hub (Transport Only)
Managed and serverless:

**Option A (Recommended for pub/sub)**: Managed MQTT (e.g., AWS IoT Core or similar)
**Option B (If you prefer routing control)**: Managed WebSocket (API Gateway + minimal fan-out Lambdas)

The authority for rules/timing is handled by the organizer (not the hub). This simplifies replays and avoids depending on a "stateful" backend.

### MQTT vs WebSocket (When to choose what)

**MQTT (IoT Core / EMQX Cloud / HiveMQ Cloud)**:
- ✅ Native pub/sub, topics per race/room, QoS and retained for "current state", LWT for presence
- ✅ Broadcast to N clients very easy and cheap; reconnection and backpressure well resolved
- ✅ Ideal for frequent positions and short events
- ➕ Retained in `race/{id}/state` simplifies "late joiners"
- ➖ Less flexible if you wanted sophisticated routing logic in between (though you can add a Lambda subscribed to topics for validations/optional relay)

**WebSocket (API Gateway)**:
- ✅ Simple full-duplex, total control of payload
- ✅ Easy to insert Lambda for inspection/anticheat in the path
- ➖ You must implement the room system, retries, presence, etc.
- ➖ Broadcast to many clients is more work (connectionId management)

**Recommendation**: MQTT for races (positions/events → pub/sub), and if someday you need complex directed/filtered messages, add a "router" Lambda.

### Topic Schema (MQTT)
```
race/{raceId}/control        # organizer commands (START, PAUSE, ABORT, SYNC_T0)
race/{raceId}/presence       # LWT and join/leave
race/{raceId}/checkpoints    # checkpoint pass events (from clients)
race/{raceId}/pos/{playerId} # compressed position (optional if only using checkpoints)
race/{raceId}/chat           # optional, room messages
race/{raceId}/state          # retained: race metadata (track, checkpoints, T0, participants)
```

**QoS**:
- `control`, `state`, `checkpoints`: QoS 1 (at least once)
- `pos/*`: QoS 0 (better latency; occasional loss doesn't matter)

**Retained**: `state` (and optionally last position per player if you want "ghosts" on reconnect)

**LWT**: Publish `{"playerId":"X","status":"disconnected"}` in presence if a client drops.

### Message Schema (Minimal JSON)

**state (retained, published by organizer)**:
```json
{
  "raceId": "abc123",
  "trackId": "desert_beetle",
  "checkpoints": [{"id":1,"x":...,"y":...,"z":...,"r":8.0}, ...],
  "t0": 1724312345.123,  // epoch seconds of START
  "serverClock": 1724312300.000, // for prior sync
  "participants": [{"playerId":"p1","name":"Guillermo"}, ...]
}
```

**control (organizer → all)**:
```json
{"cmd":"SYNC_T0","serverClock":1724312300.000}
{"cmd":"START","t0":1724312345.123}
{"cmd":"PAUSE"} 
{"cmd":"ABORT","reason":"manual"}
```

**presence**:
```json
{"playerId":"p1","event":"join","clientClock":1724312299.500,"ver":"1.2.0"}
{"playerId":"p1","event":"leave"}
```

**checkpoints (client → all/host)**:
```json
{
  "raceId":"abc123",
  "playerId":"p1",
  "cp": 5,
  "ts": 1724312355.842,      // local monotonic time relative to t0 or epoch
  "pos": {"x":...,"y":...,"z":...}, // optional
  "speed": 21.3               // optional, for heuristics
}
```

**pos (optional stream for "live" replay/ghosts)**:
```json
{"playerId":"p1","t": 10.432, "x":...,"y":...,"z":..., "v":21.3}
```

**Note**: In client store monotonic timestamps (monotonic clock) and also send epoch for cross-synchronization.

### Time Synchronization

**NTP Synchronization**: Both clients and server synchronize their clocks with an NTP server to ensure accurate UTC time across all systems.

**Race Start Timing**: The organizer sends a `t0` timestamp that represents the **future date and time** when the race will start. This timestamp is always in the future relative to the current time.

**Countdown System**: 
- Clients receive the `t0` timestamp and display a **countdown timer** on their screens
- The countdown shows the time remaining until race start based on their synchronized local clocks
- When the countdown reaches zero (local time matches `t0`), the race automatically begins

**Clock Offset Calculation**: 
- Clients calculate the offset between their local time and the organizer's reference time
- This offset is used to convert local timestamps to race time for accurate checkpoint and position reporting
- The system uses monotonic clocks to avoid drift issues while maintaining synchronization with the NTP reference

**Race Time Calculation**: Race time = (local monotonic time - local monotonic time at t0) + offset

### Anticheat/Validation (Lightweight, no stateful backend)

**In client (prevention)**:
- Only emit checkpoint if entering radius r of checkpoint in order
- Calculate speed and discard jumps > threshold (teleports)
- Basic payload signature (HMAC with ephemeral raceKey distributed by organizer on join)

**In organizer (verification and logging)**:
- Recalculate order and times with received stream
- Reject events implying impossible speeds/invalid order
- Mark suspicious ones for post-tournament review

(If you wanted an optional cloud layer: a Lambda subscribed to topics can do heuristics and publish "flags" to moderation topic without breaking the model)

### Replay Format (Local in organizer)

One file per race, e.g. `race_abc123.replay.jsonl.gz`:

One line per event (control, presence, checkpoint, pos).

**Compression**: gzip/zstd

**Delta encoding** for positions (quantize to int16 offsets and send every N ms)

**Indexes**: every N seconds write a keyframe for fast jumps in replay UI

With this you easily generate: "ghosts", heatmaps, sector times, etc.

### Security

- **TLS always** (mqtts / wss)
- **Auth per race**: token/JWT read-only/write limited to `race/{raceId}/#`
- **Ephemeral keys**: organizer creates a raceKey (short duration) on start
- **Rate limiting**: in client (throttle), and in hub (per connection) if possible
- **LWT/presence** for state cleanup

### Concrete Technologies

**Client (overlay)**:
- Blish HUD module (C#, MIT): in-game UI, integrated MumbleLink input
- Or Python + mmap/ctypes for MumbleLink, paho-mqtt or websockets, and minimal GUI (PySide6)

**Hub (serverless)**:
- **Managed MQTT**: AWS IoT Core / EMQX Cloud (free plan) / HiveMQ Cloud
- **Managed WebSocket**: AWS API Gateway WebSocket + Lambda (only if you prefer custom routing)

**(Optional) Auxiliary cloud without persistent state**:
- Lambda subscribed to topics for soft metrics/anticheat
- S3 to publish static scoreboard post-race (if you want to share online)

### Race Flow

1. Organizer creates raceId, defines checkpoints → publishes state (retained) and SYNC_T0
2. Players subscribe, do join (presence) and preheat clock offset
3. Organizer sends START with t0
4. Clients emit checkpoint (QoS1) and (optional) pos (QoS0)
5. Organizer validates + records everything
6. ABORT or finish → generates results and replay from local log

### Next Steps

If you want, in a next step I can provide:
- Complete JSON schema for message validation with ajv/jsonschema
- Client code templates (Blish HUD and Python) and organizer recorder (writer .jsonl.gz)
- Example configuration for topics/ACLs for IoT Core/EMQX

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
- **Local storage**: Complete race data stored locally for offline analysis
- **Advanced replay**: Video-like playback with interactive map and time controls
- **Data export**: Multiple formats for external analysis and sharing

---

## ✅ Advantages of this Model

- **Organizer has full authority** → can generate replays, anti-cheat, results without relying on cloud.  
- **Hub is stateless** → cheap, scalable, replaceable.  
- **Clients stay simple** → only gather MumbleLink data + send.  
- **Open-source friendly** → client (Python/Blish HUD), server (Python/C#/Electron), hub (standard MQTT/WebSocket).
- **Local data ownership** → complete control over race data and replay functionality
- **Offline capability** → replay and analysis available without internet connection
- **Data privacy** → sensitive race information never leaves the organizer's machine

---

*This system provides a comprehensive solution for organizing and managing competitive racing tournaments with real-time tracking, local data storage, and advanced replay analysis capabilities.*

