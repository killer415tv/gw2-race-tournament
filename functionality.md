# GW2 Tournament System - Functional Documentation

## Overview
This document describes the functionality of a tournament system designed for Guild Wars 2 racing events. The system consists of two main applications: a **Client** application that runs on each racer's computer, and a **Server** application that manages the tournament.

---

## 🏁 Client Application

### Purpose
The client application is launched on each racer's computer to participate in the tournament.

### Startup Operations
1. **NTP Synchronization**
   - Syncs with NTP server to get accurate UTC time
   - Ensures all clients have synchronized timers

2. **Server Connection**
   - Prompts user for server code via popup
   - Prompts user for mode: Spectate or Racer
   - Establishes connection with tournament server
   - The client does not have to set the map

3. **Status Monitoring**
   - Sends ready status to server every second
   - Measures ping to server (optional feature)
   - Sends ping values every 10 seconds

### Communication Protocol

#### **Incoming Commands (from Server)**
| Command | Format | Description |
|---------|--------|-------------|
| `msg` | `msg "message"` | Displays message on top of screen |
| `start` | `start "timestamp"` | Receives exact race start timestamp |
| `ranking` | `ranking "json"` | Receives current race ranking |

#### **Outgoing Commands (to Server)**

- **Position Updates**: Sent every 100ms
  - Only for Racers, spectators does not send any position, only receive msg start and ranking messages
  - Format: `pos x,y,z,speed,utc_timestamp`
  - Contains: 3D coordinates, speed, and internal UTC timer

---

## 🎮 Server Application

### Purpose
The server application manages the entire tournament, providing a comprehensive interface for tournament organizers.

### Startup Operations
1. **Tournament Setup**
   - Opens tournament management window
   - Collects tournament information:
     - Tournament name
     - Map selection
     - Server code

2. **Stage Initialization**
   - Sets up 3 tournament stages
   - Manager starts at Stage 1

### Tournament Stages

#### **Stage 1: Pre-Race** 🚦
- **Client Management**
  - Monitor client readiness status
  - View connected racers
  
- **Communication**
  - Send messages to all racers or specific users
  - Broadcast announcements
  
- **Race Configuration**
  - Set number of laps (default 1)
  - Set time to start after the button start race is pressed (default 1 minute)
  - **Start Race** button to initiate competition, this button sends the actual time plus 1 min to all clients, and clients enable a countdown to start

#### **Stage 2: Active Race** 🏃‍♂️
- **Access**: Available even before race starts
- **Real-time Monitoring**
  - Interactive map showing all racer positions
  - Live ranking with checkpoint information and timers
  
- **Checkpoint System**
  - Server validates when racers reach checkpoints
  - Records client timestamps for ranking calculations

#### **Stage 3: Post-Race** 🏆
- **Access**: Available even before race completion
- **Results & Analysis**
  - Complete final ranking with all checkpoint timers
  - Race replay functionality on interactive map
  - Historical position data playback throughout the race

---

## 🔄 System Flow

```
Client Startup → Server Connection → Ready Status → Race Start → Position Updates → Checkpoint Validation → Final Ranking
```

## 📋 Key Features

- **Real-time synchronization** via NTP
- **Live position tracking** with 100ms updates
- **Interactive mapping** for race visualization
- **Checkpoint-based ranking** system
- **Race replay** functionality
- **Multi-stage tournament** management
- **Real-time communication** between server and clients

---

*This system provides a comprehensive solution for organizing and managing competitive racing tournaments with real-time tracking and analysis capabilities.*

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

The organizer is the source of truth for time: sends SYNC_T0 and then START with an absolute t0.

Clients calculate offset (serverClock - clientClock) and convert their marks to race time (= clientMonotonic - clientMonotonicAtT0).

Avoid depending on system clock with drift: use local monotonic clock and only align with offset at t0.

### Anticheat/Validation (Lightweight, no stateful backend)

**In client (prevention)**:
- Only emit checkpoint if entering radius r of checkpoint in order
- Calculate speed and discard jumps > threshold (teleports)
- Basic payload signature (HMAC with ephemeral raceKey distributed by organizer on join)

**In organizer (verification and logging)**:
- Recalculate order and times with received stream
- Reject events implying impossible speeds/invalid order
- Mark suspicious ones for post-tournament review

(If you wanted an optional cloud layer: a Lambda subscribed to checkpoints can do heuristics and publish "flags" to moderation topic without breaking the model)

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

