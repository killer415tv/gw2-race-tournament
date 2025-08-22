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

