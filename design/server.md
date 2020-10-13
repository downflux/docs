---
layout: default
title: DownFlux Networking Design
---

# DownFlux Networking Design
Client-Server Model for a Large-Scale RTS

| Status         | draft                 |
| :------------- | :-------------------- |
| Author(s)      | minke.zhang@gmail.com |
| Contributor(s) |                       |
| Last Updated   | 2020-10-09            |

## Objective

Design a communications model between a small number of clients concurrently
mutating a complex RTS game state.

## Background

Relevant to this document, DownFlux will be an RTS game for a small number
(~10) of players within normal RTS parameters. We are exploring different
networking models for the game, and have proposed the following design as a
potential implementation of client-state interaction.

## Overview

### Assumptions

| Metric                | Estimated Bound                                                                                                                    |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Players               | 10                                                                                                                                 |
| Controllable Entities | 1k                                                                                                                                 |
| Map Size              | 1k x 1k tiles                                                                                                                      |
| Server Tick Rate      | 10Hz (see [justification](https://www.reddit.com/r/starcraft/comments/8q0jka/e0fhawd?utm_source=share&utm_medium=web2x&context=3)) |
| Network Latency       | 100ms                                                                                                                              |
| Network Bandwidth     | 1Mbps / player (see [justification](https://gamedev.stackexchange.com/a/96184))                                                    |

## Related Work

A list of historically relevant papers and articles can be found in the
[DownFlux docs repo](https://github.com/downflux/docs/blob/main/brainstorm/networking.md).

## Infrastructure

The networking model will be a client-server model, as opposed to the more
commonly implemented lockstep framework used in most RTS games. We've decided
that this should dramatically simplify the design, and dodge around tricky
issues like creating a consistency model for a fully connected P2P system from
scratch. One of the main benefits of the lockstep model is saving on
computation and bandwidth costs (on the centralized server)<sup>1</sup>, but
modern consumer processing power and bandwidth should be enough to handle the
workload.

The client consists of a renderer<sup>2</sup> and an API component, with the
API component sending player commands to the server, e.g. "build barracks
here", "attack enemy infantry", "use special ability", etc.

The server consists of two message queues (input and output), and a core loop
which handles the burdensome task of simulating the game state. The input queue
regularly issues a list of player-issued messages to the core loop, sorted by
the server clock time. The core loop then takes these messages and runs through
an internal subprocess order in which these messages and the existing game
state are taken as input. A list of output messages will be then piped to the
outgoing queue, which will fire the back to the client immediately, along with
information on when the messages should be rendered. The client will merge
these messages into the existing game state to be picked up by the rendering
component.

## Detailed Design

### Types

```
type ClientID, EntityID, CurveID string

type TickID string
type Tick float64

type Entity interface {
    ID() EntityID
    Curve(t (HP|PRIMARY_COOLDOWN|...)) CurveID
}

type Curve interface {
    ID() CurveID
    Type() (LINEAR|STEP|PULSE|...)
    Parent() EntityID
    Tick(float64) Tick
    Value(Tick) float64
}
```

### Client

```
CURRENT_TICK_ID TickID = ""
```

#### API Component

Breaking down the command that will be issued to the server, we can broadly
speculate this would include

* `Build(c Coordinate, t (BARRACKS|REFINERY|...))`
* `Ping(c Coordinate, t (ATTACK|GUARD|...))`
* `Move(entity_id string, c Coordinate, t (NORMAL|REVERSE|ATTACK_MOVE|GUARD|...))`
* `Attack(entity_id, target_entity_id string)`
* `UseAbility(entity_id, target_entity_id string, c Coordinate, t (PRIMARY|SECONDARY|ULTIMATE))`

Each API call will add an additional `CURRENT_TICK_KEY` to each message when
sending the message to the server.

#### Server

The server simulates the entire game state and facilitates player-player
interaction (e.g. combat). At the heart of the server is a linear game loop
consisting of _phases_. Phases are run serially, but logic within each phase
should exploit concurrency where possible.

```
SERVER_TICK_RATE int = 10
CURRENT_TICK Tick = 0  // At 60Hz, we will need to run the game for
                       // 400+ days before encountering overflow
CURRENT_TICK_ID TickID = ""
TICK_ID_LOOKUP map[Tick]TickID = nil  // length TICK_ID_WINDOW_SIZE
TICK_ID_WINDOW_SIZE int = 2  // Number of ticks in the past the
                             // server will accept as valid (initial)
                             // input
```

#### New Tick Phase

This phase is a trivial subroutine which

1. increments `CURRENT_TICK`,
1. generates a new `CURRENT_TICK_ID`, and
1. update `TICK_ID_LOOKUP` with the new data, as well as dropping the oldest
   row

#### Input Queue Phase

```
CLIENT_RECENT_TICK map[ClientID]Tick = nil
MESSAGE_QUEUE []PlayerCommand = nil
```

The input phase keeps a buffer of incoming player messages. The buffer is
sorted by the received timestamp of the server.

For each incoming message, the input phase will do some basic precondition
tests:

1. if message has an embedded `TickID` which does not show up in the keys of
   `TICK_ID_LOOKUP`, discard message and relay error to sender;
1. then, if message has an embedded `TickID` corresponding to a `Tick` _before_
   the `Tick` found in `CLIENT_RECENT_TICK`, discard message and relay error to
   sender;
1. then update the client's `CLIENT_RECENT_TICK` entry with the corresponding
   `Tick` and enqueue the message

At the **beginning** of each tick, the queue will

1. discard duplicate messages,
1. log queue alongside `CURRENT_TICK`,
1. sent off to the core loop for processing.

#### Output Queue Phase

```
MESSAGE_QUEUE []Curves = nil
```

The output queue keeps a buffer of outgoing messages to each player. An
outgoing message may either be an entity mutation (e.g. a creating or
destroying a building or unit), or a curve<sup>6</sup> mutation (e.g.
altering a path, starting an attack, etc.).

For each player, we will need to

1. filter the outgoing messages by the player POV, i.e. don't broadcast
   stealthed units or units under the fog of war, and
1. filter any curve message by the player POV, i.e. don't show a position curve
   (movement) that goes 50 ticks in the future if the position of the unit will
   exit the player POV; we can optionally skip this step if the domain of the
   curve extends into the future only a little bit (e.g. less than 1s of
   rendering time)

This filter step can be a no-op for now while we implement everything else, but
if we do not enable filtering, the player can exploit the additional
information in the form of map hacks.

At the **end** of the tick, the output phase will

1. log unfiltered queue with `CURRENT_TICK`,
1. send players their respective messages, along with the new `CURRENT_TICK_ID`

#### Core Loop Phase

```
TRIGGER_QUEUE map[Tick]CurveID = nil
ENTITY_LOOKUP map[EntityID]Entity = nil  // Entity.Curves() is a list
                                         // of CurveIDs
CURVE_LOOKUP map[CurveID]Curve = nil  // Curve.Parent() is a single
                                      // EntityID
```

The core loop is tasked with updating the actual game state, including editing
the map terrain and mutating the curves. This is a heavy subroutine.

##### Delete Entities

For the current server tick, we check the `TRIGGER_QUEUE`<sup>7</sup> for any
curves that have significant effects, e.g. setting health to 0. For these
curves, we need to

1. delete the parent entity (e.g. structure or unit)
1. delete the row from the queue

##### Create Entities

New structures and units may be instructed to be built, either by a player
command (new structure) or when a production facility finishes production (unit
ready). For the former, we will read from the message queue, whereas the latter
will need to be checked against `TRIGGER_QUEUE`.

New entities will need to be added to the `ENTITY_LOOKUP` master list.

For buildings which have a set construction time, we will

1. generate a new curve for the new entity representing when the structure may
   be used (e.g. start producing units),
1. add the curve to the output queue

##### Update Curves

Each subphase of curve mutation may be done concurrently.

###### Collision Detection

We need to check if any entities (units, buildings, projectiles, crates, etc.)
overlap hitboxes -- if they do, we need to resolve any actions that may occur
(taking damage, grabbing upgrade, redo pathing, etc.). For entities which
require deletion (e.g. projectiles and crates), add to `TRIGGER_QUEUE` -- do
not delete in this step.

Collision detection may be implemented via
[QuadTree](https://gamedev.stackexchange.com/q/48565), but can be null-op for
now. This phase may possibly support being done asynchronously by a separate
process in the background.

###### Pathing

For this phase, we will read from the message queue and update / create curves
with new paths (and add to the output queue). This may

* be done in parallel, and
* is abstracted away from the actual implementation of pathfinding and may be
either generated by HPF<sup>3</sup> or flow fields<sup>4</sup>.

If flow fields are used for pathfinding, the simulation for steering forces are
all simulated on the server.

###### Attack Resolution

Attacks from the input queue will need to be processed and aggregated. For each
attack command from the queue, we will need to find the target entity and
connect the curve with the target entity. This will be in the form of an HP
curve for the target entity, which itself is an aggregated curve formed by the
summation of all attack (and heal) curves targeting the entity.

If the HP curve has changed, add / update the `TRIGGER_QUEUE` row for when the
HP curve reaches 0

Any updated curves will need to be added to the output queue.

##### Ability Resolution

Abilities like shields, speed boosts, etc. will have associated curves and must
be updated and added to the output queue.

## Caveats

### Tick Rate Tuning

The server will have significant impact on the network latency in this model --
if we assume (per [Assumptions](#assumptions)) a 100ms client-server travel
time, and that the server itself will take another 100ms (at 10Hz), our total
end-to-end latency is 300ms. While it may not matter much ultimately in the
game results<sup>5</sup>, we will still need to employ around 20 frames of
[client-side prediction](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
to smooth out user input. How fast can we make the server tick rate to cut down
on the minimal latency?

### Client API Component

We could alternatively consolidate `Move`, `Attack`, and `UseAbility` into more
general API endpoints:

* `EntityTargetAbility(entity_id, target_entity_id string, t (MOVE|REVERSE|ATTACK_MOVE|ATTACK|PRIMARY|SECONDARY|...))`
* `EntityTargetAoE(entity_id string, c Coordinate, t (GUARD|PRIMARY|SECONDARY|...)`

We decided this is needlessly generic and will create more problems server side
in decoding the intent of the API than it solves in a unified API.

## Scalability

### Multi-Server Processing

Look into Redis for in-memory SQL implementation, as our `Event` and `Curve`
data are rather tabular (as are the message queues). This is useful if / when
we scale up to multiple server nodes.

## Redundancy and Reliability

TBD

## Security

TBD

## Privacy

The server will be aware of

* identifiable user data (IP, username, etc.)
* user game input (game commands) and the time the commands were received

The server will not keep track of the user IP, or other non-game related
identifiable data. The user ID, username, and game input will be tracked for
replay purposes.

## Work Estimates

| Work Item                   | Time Estimate            | Status      |
| --------------------------- | ------------------------ | ----------- |
| barebones client and server | 1 week                   | NOT STARTED |
| implement tick phase        | 1 day                    | NOT STARTED |
| implement input queue       | 1 week                   | NOT STARTED |
| implement output queue      | 1 week                   | NOT STARTED |
| implement create entities   | 1 week                   | NOT STARTED |
| implement pathfind          | 1 week                   | NOT STARTED |
| ...                         | lifetime of the universe | NOT STARTED |

## Footnotes

<sup>1</sup>Terrano, Mark. "1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond." 2005.

<sup>2</sup>Discussing the specifics of the rendering engine is out of scope of
this design document.

<sup>3</sup>Botea, Adi. "Near optimal hierarchical path-finding." 2004.

<sup>4</sup>Emerson, Elijah. "Crowd Pathfinding and Steering Using Flow Field Tiles." 2020.

<sup>5</sup>Claypool, Mark. "The effect of latency on user performance in Real-Time Strategy games." 2005.

<sup>6</sup>Stolen from
[The Tech of Planetary Annihilation: ChronoCam](https://www.forrestthewoods.com/blog/tech_of_planetary_annihilation_chrono_cam/).
Curves are linear transformations of a variable trajectory. This transformation
saves on data being sent to the client.

<sup>7</sup>We need to decide if we want a generic trigger queue, or a queue
broken down by category, with the `CurveID` still mapping back to a global
lookup map.

