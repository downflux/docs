# Commanding RTS Commands
Scaling State Mutations via FSM Visitors

_DownFlux is a real-time strategy game in active development at
[github.com/downflux](https://github.com/downflux). I have several years of
professional software development experience, but no experience with game
development specifically. This document doesn't espouse a general solution
applicable to all problems, but just demonstrates potentially an alternative to
command patterns for a narrow set of circumstances. For a more technical
overview of this approach, take a look at the [design
doc](https://docs.downflux.com/design/fsm.html) instead._

_I also liberally use the third person plural here because it sounds awkward to
say I all the time. Apologies._

## Background

One of the many challenges I've come across working on
[DownFlux](https://github.com/downflux/game) has been to design a flow framework
which affords the development of a great many state change operations without
tying everything down into a gigantic smelly mess as we add more and more
commands. This article discusses an approach that we've found to work for us --
maybe you'll find something worth using in your own efforts.

## Jargon

* state mutations, flows, commands -- a series of changes to the game state
  (e.g. map, entities, etc.) which achieve a specific end-goal (e.g. `move`)

### Flow Examples

* `move(source, dest)`: move the source object to the destination location.
* `chase(source, target)`: series of serialized moves, which are updated as the
  destination object moves.
* `attack(source, target)`: chase target asynchronously; if target is within
  attack range and the source can attack (off cooldown), then commit state
  change.

## An ad hoc Approach

Our first attempt at mutating the state was pretty simple, as we only
implemented the `move` command; we decided on a basic command pattern, where on
each tick, the server will run through all installed commands and call a simple
API.

```
func (s *Server) Run() {
  for {
    for c := range s.Commands {
      // Client calls mutate this CommandQueue object by appending
      // pending commands.
      c.Execute(s.CommandQueue[c.Type()])
    }
  }
}

type Command interface {
  Execute(args interface{}) error
  Type() CommandType
}

func (c *MoveCommand) Execute(args interface{}) error {
  a = args.(MoveCommandArgs)

  // p is a list of Position objects (i.e. (x, y) tuples).
  p = c.Map.GetPath(a.Source.Location.Get(a.Tick), a.Destination)

  // Source merges the positions with internal velocity in
  // the curve.
  a.Source.Location.Update(p)
  return nil
}
```

<a name="figure-1">Figure 1</a>: Simple implementation of the `move` command.

Simple!

Our first hiccup is the fact that `GetPath` can be expensive if we were to
report the full path -- these computing cycles may be wasted if the user
instructs the same source object to move to a different location before the
source reaches the destination,[^1] as is often the case in RTS games.
Therefore, we want to calculate and set a short trajectory, with delayed
execution of the rest of the path (see [Figure 2](#figure-2)).

![Partial Move DAG](assets/scaling_commands_partial_move.png)

<a name="figure-2">Figure 2</a>: Partial path diagram. The command should only
calculate p<sub>0</sub> first; at some time t in the future, recalculate the
path (which may involve further sub-path iterations).

With the partial path logic, our command now looks something like this:[^2]

```
func (s *Server) Run() {
  for var c := range s.Commands {
    c.Execute(curTick)
  }
}

// Called by client API as well as internally.
func (c *MoveCommand) Schedule(
  t Tick,
  e Entity,
  d Destination) error {

  scheduledAction := c.q.Get(e)
  if scheduledAction != nil && scheduledAction.Precedence(t) {
    c.q.Set(t, e, d)
  }
}

func (c *MoveCommand) Execute(t Tick) error {
  const pathLen int = 10;

  for MoveCommandArg arg := range c.q {

    // Return a path of a specific length instead.
    p = c.Map.GetPath(
      arg.Source.Location.Get(t),
      arg.Destination,
      pathLen,
    )

    arg.Source.Location.Update(p)

    if p[len(p) - 1] != arg.Destination {
      c.Schedule(
        t + a.Source.CalculateTravelTime(p),
        arg.Source,
        arg.Destination)
    } else {
      c.Delete(arg)
    }
  return nil
}
```

<a name="figure-3">Figure 3</a>: Toy `move` command implementation v2 -- here we
enqueue a delayed move command into the main command queue. This queue may have
client- or other server-initiated command scheduling, so when we update the
queue, we need to ensure there is a single, canonical execution flow.

Kind of a pain, but still doable.

This model worked well enough for us to get a rudimentary frontend client
running that can issue the move command to the server, and the game state update
appropriately.

However, a gut check seems to indicate major scalability issues with this
approach[^3]. In particular,

* If there are state changes to the object which impact the command, e.g. if the
  source is within attack range, it is up to the command to manually check.

* The `Execute` function needs to keep track of the state of _all_ dependent
  scheduling queues (similar to `c.q`) however, if a command may mutate multiple
  queues, e.g. an `attack` command potentially needing to edit the `chase`
  queue, this seems incredibly hard to manage without an overall framework or
  design guidance.

* Making the command iterate over its own schedule just seems wrong -- why is
  the command responsible for when it's called? A better separation of
  responsibility intuitively seems more elegant than what we have here.

## Tour de Entities

Looking back, it feels the driving force for the refactoring efforts we've made
in search of a solution to the overarching problem is that the command execution
cycle has too much scope and authority. We can view our subsequent changes as a
way to clamp down on the freedom of the command to arbitrarily change the
system.

One such effort is highlighted by the last concern stated above -- Why does a
command have to burden itself with scheduled iteration? The `move` command only
changes a single object -- what if we mirrored our command API to this model?

This line of questions lead us to use the _canonical_
[visitor model](https://en.wikipedia.org/wiki/Visitor_pattern) example, i.e.
where we implement `Accept` on tangible objects.

```
func (s *Server) Run() {
  for var v := range s.Commands {
    for var e := range s.Entities {
      e.Accept(v)
    }
  }
}

func (e *EntityImpl) Accept(v Visitor) error { return v.Visit(e) }

func (c *MoveCommand) Visit(e Entity) error {
    if c.q.Has(e) {
      // This is the same implementation as in [Figure 3](#figure-3).
      c.Execute(c.Status.CurrentTick())
    }
}
```

<a name="figure-4">Figure 4</a>: The architectural change counterpart to the
changes made in [Figure 3](#figure-3). The key takeaway is that our "`Execute`"
function (`Visit`) no longer takes in an `interface{}` input -- we are starting
to discourage implementing commands with arbitrary ease, with the return of a
slightly more sensible execution model.

...Okay so this doesn't seem to be very different from our ad hoc approach
(yet). Bear with me as we move on.

## Finite State Metadata

IMO, a large part of the code smell at this point is due to our scheduling
management -- it seems like an exceedingly poor choice to make the flow -- the
logic used to mutate the _entity_ -- be in charge of mutating _itself_. How can
we decouple these two things?

A complementary issue is suggested by the command API:

```
  c.Visit(e Entity)
```

This is clear when our command mutates a single object like `move` -- but what
happens when we need to mutate multiple objects like `attack`? It seems rather
naive to delegate a "more" important object here in our visitor model, e.g.
`AttackCommand.Visit(attacker)` and hiding the target in the corresponding
schedule, but it also seems overly complex to split this command into an
`attack` and `recieve_damage` command.

The solution to both of these problems indicate that our scheduler is probably
complex enough to warrant additional development cycles.

As may be suggested by the section header, the next leap of logic we made was to
consider the scheduler rows as their own important [meta]data objects. These
objects don't really describe the underlying state of the game (e.g. the
position of objects); rather, they describe the state of the state _mutations_.
We already encountered this of sorts in [Figure 3](#figure-3):

```
type MoveCommandArgs struct {
  scheduledTick Tick
  source        Moveable
  destination   Position
}
```

<a name="figure-5">Figure 5</a>: Expanded `MoveCommandArgs` type from
[Figure 1](#figure-1).

One key observation is that we know the `move` command has finished if

```
source.Location.Get(currentTick) == destination
```

Can we model the other states of the `move` command with similar tests?

(Yes.)

Let's consider the state diagram of the `move` command first:[^4]

![Move DAG](assets/scaling_commands_move_dag.png)

<a name="figure-6">Figure 6</a>: `move` state diagram.

* `FINISHED`: As stated above, we know a command is finished if the source has
  arrived at the destination at a specific tick.
* `PENDING`: If the internal scheduled "tick" does not equal the current tick,
  the command is not yet ready to execute; this accounts for both when the
  source is already moving, or still needs to calculate the next partial move.
* `EXECUTING`: If the internal tick equals current game tick, we will need to
  calculate the path of the object. At the end of the execution phase, the
  scheduled tick should be updated.
* `CANCELED`:[^5] An externally triggered transition if e.g. the client
  specifies another move command in the meantime.

Remember that the `MoveCommandArg` struct was used in the original command
implementation model -- we've demonstrated previously that there is enough
information in the struct to allow the command to execute. Because this is the
only data that matters to the command, let's iterate over that instead.

```
func (s *Server) Run() {
  for var v := range s.Visitors {
    for var q := range s.CommandQueue[v.Type()] {
      q.Accept(v)
    }
  }
}

func (v *MoveCommand) Visit(m MoveCommandArgs) {
  s := m.Status()
  switch s {
  case EXECUTING:
    ...
    m.SetNextPartialExecution(m.Status().Tick() + ...)
  default:
    return
  }
}
```

<a name="figure-7">Figure 7</a>: Final iteration of the command structure.

On the surface, this honestly doesn't look that different from
[Figure 1](#figure-1)! So why make this journey at all then?

There are several things this FSM - Visitor approach addresses which were
problematic in our ad hoc solution:

* > If there are state changes to the object, it is up to the command to
  > manually check.
  Because the command _metadata_ does state checking for us, and because we know
  the check is idempotent since the underlying game state is deterministic
  barring user-input,[^6] the execution model of the command is greatly
  simplified.
* > The `Execute` function needs to keep track of the state of _all_ dependent
  > scheduling queues
  The game state accessible by the command execution logic is scoped
  specifically by the metadata -- no extra data is being surfaced, lowering our
  risk of wanton data access.
* > Making the command iterate over its own schedule just seems wrong.
  The execution logic behavior is solely dependent on the metadata; no
  information about the schedule at large is leaked.
* Additionally, because we are instead mutating and iterating over the metadata,
  we have avoided entirely the semantic issue of forcing an entity into our input.

As a nice side-effect, because the metadata object acts as a gamestate proxy
object for the command itself, we no longer need to create a fully initialized
game state when testing a specific command -- it is sufficient to populate the
_proxy_ with a reasonable subset of the state. Empirically, this has greatly
decreased the barrier to unit testing, which naturally increased the number of
tests written for these commands.

Let's apply the same pattern to the `attack` command, a flow which has a
dependent `chase` action.

```
type AttackMetadata struct {
  s     CanAttack
  t     CanDie  // Mortal?
  chase *ChaseMetadata
}

func (m *AttackMetadata) Status() Status {
  if chase.Status() == CANCELED { return CANCELED }
  if t.Health(curTick) <= 0 {
    return FINISHED  // Cleaned up next tick.
  }
  if d(s, t) < s.AttackRange() && a.OffCooldown(curTick) {
    return EXECUTING
  }
  return PENDING
}

func (c *AttackCommand) Visit(m AttackMetadata) {
  if m.Status() == EXECUTING {
    t.Damage(a.Strength())
  }
}
```


<a name="figure-8">Figure 8</a>: Simplified `attack` command implementation.
**Note that the `AttackMetadata.Status` function is read-only** -- this is by
design. The metadata object can only mutate the game state if the command
executor explicitly calls a mutate endpoint. See the design doc for more
information.[^7]

Dependencies in our framework are modeled by a pointer in the metadata to
another metadata object. In doing so, we are making an explicit statement in
code that there is a clear dependency; because the dependent structure is
formally defined through interfaces, there is a grammar we can use to talk about
these dependencies. In particular, we are hiding the specifics of the dependency
from the actual mutation -- to the command executor, this dependency is actually
just an implementation detail of the metadata, and can therefore be ignored.
This decoupling of state reads from state writes is crucial for our scalability
requirements.

Also note that the metadata struct does not mutate itself -- the visitor has to
explicitly change the FSM state. Because our server implementation iterates over
all visitors per tick, we can be sure that any concurrent changes are made
_only_ by multiple calls to the same `v.Visit()` function. This greatly
simplifies the mental execution model.

So far, we have found this approach to be quite flexible and sufficient for our
needs; you can browse our
[repo](https://github.com/downflux/game/tree/8fbaefebcb31d5f59796c6285595ccda544dc02f),
snapshotted at the time of writing, to see how we actually implement these
flows. Take a look and let me know what you think!

## See Also

* [Arbitrary Command Execution](https://docs.downflux.com/design/fsm.html)
  Technical design doc of this approach that goes deeper into implementation
  specifics.
* [Time-Invariant Finite State Machines](https://blog.kevmo314.com/time-invariant-finite-state-machines.html)

## Addendum

### Recontextualizing as Event Flows

I came across a rather interesting tech talk while writing this article which
talks about the
[Event-carried State Transfer](https://youtu.be/STKCRSUsyP0?t=896) software
pattern (indeed, from what little research I've done on this, it seems like this
talk is actually _the_ talk which introduced the concept to the wider public).

There are some interesting parallels here between the event-driven approach
described and ours here. Indeed, when the command executors branches on the
metadata state, we're effectively detecting if an event occurred between the
last and current server tick! Additionally, the event-carried state transfer
pattern seems to revolve around **minimizing data access to the underlying
state** -- the event pattern achieves this through some level of caching packed
into the event data in order to reduce resource contention, whereas we are
minimizing the API surface area that is exposed through the command metadata.
Maybe this is an example of convergent software evolution.

It is true that we could massage our current approach into an event-driven
approach; however, this seems both overengineered and antithetical to how we
view our code.

1. Remember that we are treating the game system as deterministic when an object
  moves, the partial move schedule is already preordained -- there is no
  additional user input that is necessary in order to make the system behave
  correctly. Our framework accounts for this by doing a series of state reads.
  However, if we were to transform the state transitions into broadcasted
  events, we're essentially advocating that the system is always in flux, and
  we're "promoting" deterministic behavior into the category of "unexpected"
  inputs. This seems like a less elegant approach, and at the same time will
  require a large system overhaul for questionable value.
1. A single server tick will execute a list of commands in a known order, e.g.
  we process all `move` commands, then all `attack` commands, etc. Event queues
  are very useful when we are decoupling execution _order_ from our server;
  however, if we were to do this, then a whole new, scary world of consistency
  problems appear. We can leave that problem to concurrent text editors and
  CRDTs.


### [A Digression on Attacking](#a-digression-on-attacking)

While editing this document, a [friend](https://www.jonkimbel.com) pointed out
that the toy implementation of the `attack` command does not fully specify some
edge-case behavior --

> I know you said this is simplified, but how do you handle situations where the
> command calls for a stationary source (e.g.
> [tesla coil](https://cnc.fandom.com/wiki/Tesla_coil_(Red_Alert_2))) to attack
> a target which then leaves its range?
> 
> Does it stay in the command queue in case the target comes back into range,
> with some lower-priority "auto attack" command dealing damage to nearby
> enemies in the meantime? Or does it cancel itself?

This question demonstrates a nice property of the FSM / visitor approach, which
is the flexibility of implementation. The implementation in
[Figure 8](#figure-8) assumes that the target can move, and will always try to
attack the same target until the target dies. How do we extend this command?

We can envision an `attack` variant that forgets the target after the target
goes out of range:

```
type ForgetfulAttackMetadata struct {
  s           CanAttack
  t           CanDie
  hasAttacked bool
}

func (m *ForgetfulAttackMetadata) Status() Status {
  if t.Health(curTick) <= 0 {
    return FINISHED  // Cleaned up next tick.
  }
  if d(s, t) < s.AttackRange() && a.OffCooldown(curTick) {
    return EXECUTING
  }
  if m.hasAttacked && d(s, t) >= s.AttackRange() {
    return CANCELED  // Cleaned up next tick.
  }

  return PENDING
}

func (c *ForgetfulAttackCommand) Visit(m ForgetfulAttackMetadata) {
  if m.Status() == EXECUTING {
    t.Damage(a.Strength())
    m.SetHasAttacked()
  }
}
```

<a name="figure-9">Figure 9</a>: Alternative `attack` command implementation.
Which cancels itself if the target exits range via a read-only operation.

### Partial Tick Execution

Because the metadata is stored in a separate queue, we can pause visitor
execution at any given time during a tick -- this means we can smooth out large
server loads over several ticks, allowing us to enforce a consistent server tick
rate (at the expense of some additional end-to-end latency). This feature is not
currently implemented in our game yet, pending load testing.

## Notes

[^1]: Partial pathfinding is implemented via
    [hierarchical A*](https://webdocs.cs.ualberta.ca/~mmueller/ps/hpastar.pdf),
    though this may / will change in the future. The point is that there may be
    additional complexity introduced into commands. As an interesting sidenote,
    partial pathfinding allows us to spread out pathfinding to multiple workers
    after the initial coarse-grain search. This may be a nice optimization route
    to go down in the future.

[^2]: In reality, this step was implemented along with initial visitor pattern
    migration (explained later), but we're highlighting a rather important
    motivating point for seeking better approaches to the problem.

[^3]: While `interface{}` inputs are undesirable, they aren't necessarily an
    _architectural_ problem. We're concerned with what are potential
    project-terminators due to non-maintainability.

[^4]: For more information on this, see
    [Time-Invariant Finite State Machines](https://blog.kevmo314.com/time-invariant-finite-state-machines.html).
    State transitions are traditionally triggered by an "external" user; we are
    expanding the FSM here to allow for the possibility that transitions may be
    triggered without an explicit outside trigger action. This allowance gives
    us a lot of flexibility in modeling semi-autonomous commands.

[^5]: Sidenote, I learned the objectively better "cancelled" spelling is
    British, and so have reverted to the inferior but semantically consistent
    American spelling.

[^6]: See [Arbitrary Command
    Execution](https://docs.downflux.com/design/fsm.html) for more details -- we
    batch incoming client API calls and merge them into the command queue at the
    beginning of each tick, so that when we actually call `Command.Visit`, we
    know the state does not have any ongoing conflicting mutations going on
    simultaneously.

[^7]: For a more in-depth discussion of the `attack` command
    implementation details, see
    [A Digression on Attacking](#a-digression-on-attacking)
