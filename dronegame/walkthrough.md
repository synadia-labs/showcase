# Drone Game

## Overview

For KubeCon NA 23, wanted to build something that would push NATS to the limits and stun the audience. So we built an Augmented Reality drone game! The game is played in a web browser and uses a 3D scene to show the drones and bullets. As users play the game, millions of data points are collected and stored both in NATS and a database. We use every part of NATS (Jetstream, KV, Object Store) to build the game data. For things that aren't a good fit (user ranking queries) we use embedded SQLite.

To play the game, just grab your phone or tablet and headover to https://dronegame.io.

You will be asked for permissions to allow access to the sensors and once you've done that, tilt and pan your phone to scope the flying drones, then tap the screen to shoot. At the end of the round, your data will be viewable and you have the option of appearing on the leaderboard.

For the general release, we removed the camera requirement, so the drones appear in an arena. This makes people less suspicious we're doing anything questionable without retracting from the game itself.

## Sequence Flow

### Match begins

Route: `/matches/new`

```mermaid
sequenceDiagram
    participant wc as 3D WebComponent
    participant b as Browser
    participant be as Backend
    participant db as SQLite
    participant js as NATS Jetstream Match Events
    participant kv as NATS KV Match State

    b->>be: Initial Page Load
    be->>b: Page with 3D WebComponent & Datastar with match data
    wc-->>b: Load 3D Scene
    wc->>be: POST protobuf sensor data (rotation, clicks)
    par
        be->>kv: Store current match state
        be->>js: Publish events from sensors + start/end/hit
        be->>db: Insert score data into Leaderboard
    end
    wc->>be: Redirect when match ends to /matches/{matchId}
```

### Match details

Route: `/matches/{matchId}`

```mermaid
sequenceDiagram
    participant wc as 3D WebComponent
    participant b as Browser
    participant be as Backend
    participant os as NATS Object Store Match Cache

    b->>be: Initial Page Load
    be-->>b: Page with 3D WebComponent & Datastar
    wc->>b: Load 3D Scene
    wc->>be: GET protobuf match data
    be->>os: Upsert match cache from match event stream
    os-->>wc: Send match objects (drones, bullets, etc)
    wc->>wc: Create animation keyframes, optimize 3D scene
    wc->>b: Report stats via Datastar to other parts of the page
```

#### Details

The match details page will show the match data and the 3D scene. The 3D can get large so there is an optimization step to reduce the number of animation keyframes.

### Match live

Route: `/matches/live`

```mermaid
sequenceDiagram
    participant wc as 3D WebComponent
    participant b as Browser
    participant be as Backend
    participant js as NATS Jetstream Match Events


    b->>be: Initial Page Load
    be-->>b: Page with 3D WebComponent & Datastar
    wc->>b: Load 3D Scene
    b->>be: Start GET /matches/live/datapoints SSE connection
    be->>js: Subcribe to live data points
    js-->>b: Send live data points as signal patches (base64 encoded strings)
    b-->>wc: Update 3D scene with signal patches (b64 -> protobuf)
```

#### Details

Multiple matches can be live at the same time. The backend will show the latest data points for a match unless it stops sending data for N seconds, then it will pick the next match message as the current one.

### Leaderboard

Route: `/leaderboards/daily`

```mermaid
sequenceDiagram
    participant b as Browser
    participant be as Backend
    participant kv as NATS KV Match State
    participant db as SQLite

    b->>be: Initial Page Load
    be-->>b: Page with Datastar
    b->>be: GET /leaderboards/daily SSE connection
    be->>kv: Subscribe to leaderboard data changes
    kv->>db: Watched matches query DB
    db-->>b: Return leaderboard data via SSE
```
