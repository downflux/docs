# DownFlux Client Design Doc
Client-Facing Engine Design

| Status       | Draft                 |
| :----------- | :-------------------- |
| Author(s)    | minke.zhang@gmail.com |
| Contributors |                       |
| Last Updated | 2020-10-23            |

## Objective

Outline the core mechanics necessary for rendering the
[server](server.md)-calculated game state to the players.

## Background

DownFlux is a collaborative RTS.

## Overview

The client will consist of two parts -- the actual rendering engine (e.g.
drawing tanks and units on screen) and the client API component that does the
lower level communications with the server.

## Detailed Design

### Client

The API client is tasked with the lower-level communications with the server,
and forwards the transformed user inputs given by the [Renderer](#renderer).
The client also runs a daemon thread to process incoming data provided by the
`StreamCurves` endpoint.

```csharp
class APIClient {
  public string ID;  // Client ID used to identify the player to the server.

  // Invokes AddClient and announces to server the current tick of the client
  // (in case of reconnection)
  public string Connect(string tickID);

  // Returns the current buffer of recieved data from server stream. Will clear
  // buffer after invocation.
  public StreamData Data;

  public void Move(
    string tickID,
    List<string> entityIDs,
    Position destination,
    MoveType moveType,
  );

  public async Task StreamCurvesLoop(string tickID);
}
```

### Renderer

The rendering engine will process actual user input (key presses, mouse
clicks, etc.), transform them into useful game intents, and forward them via
the API client to the server.

The engine will also be tasked with the core loop of processing game state and
displaying that in some form to the user.

#### Core Loop

```csharp
using EntityID = string;
using CurveID = string;

public Dictionary<EntityID, Entity> EntityLookup;
public Dictionary<CurveID, Curve> CurveLookup;

public Queue<PlayerAction> Actions;
```

##### Tick Rate

The server and rendering engine will run at differing tick rates -- because
we aim to minimize the network traffic for communicating game state, the
data sent to the client will be much slower than the rate at which information
needs to be redrawn. For example, the canonical server tick rate is at ~10Hz
(from server design), but obviously for games, the renderer needs to draw at a
rate of 30 - 60Hz, if not higher.

To account for this tick discrepancy, the renderer will interpolate the server
curve data for the relevant partial server tick times.

The renderer may also need different curve rates for different phases in the
core loop -- for example, if the server only broadcasts game state at 10Hz, the
renderer doesn't need to poll for every frame (60Hz).

###### Server Reconciliation

```csharp
public static TickRate = 10;
```

The renderer will need to query the API client regularly to update its internal
curve state and for new entity announcements. The API client holds the actual
server stream.

If any new data is provded from the API client (whether new entity
announcements or curve updates), the renderer will update `EntityLookup` and
`CurveLookup`, either with `Curve.ReplaceTail` or by creating a new entity
row.

*N.B.*: `EntityLookup` and `CurveLookup` are add only sets. If an entity should 
no longer be rendered, it will be marked as tombstoned instead.

###### Rendering

The renderer will iterate through the set of curves and draw appropriate data
for the values interpolated at the current client tick time.

The server will also need to iterate through the `Actions` queue and render any
client-only changes.

###### Process Player Input

TODO(minkezhang): Is this async?

At this phase, the renderer will take note of the current player actions (e.g.
physical clicks, mouse drags, etc.), transforms them into a usable struct, and
append them to the `PlayerAction` queue.

###### Process Player Actions

Depending on the action specified (e.g. `Move` or `ScrollViewport`), the
may need to communicate with the server -- this phase is leaving room for
calling out action-specific handlers

The renderer will call the server (via the API client) with appropriate
commands at this time.

# See Also

* [DownFlux Networking Design](server.md)




















