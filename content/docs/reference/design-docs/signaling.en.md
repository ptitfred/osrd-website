---
title: "Signaling"
linkTitle: "Signaling"
weight: 43
description: "Describes the signaling model"
---

{{% pageinfo color="warning" %}}
This document is pending review
{{% /pageinfo %}}

## Description

The signaling layer includes all signals, which respond to track occupancy and
reservation.  Signals can be of different types, and are modularly loaded. Only
their behavior towards the state of the infrastructure and the train's reaction
to signaling matters.

Signals are connected to each other by blocks. Blocks define the movements
authorized by signaling.

## Goals

The signaling system is at the crossroads of many needs:

- it must allow for realistic signaling simulation in a multi-train simulation
- it must allow the conflict detection system to determine which resources are required for the train
- it must allow application users to edit and display signals
- it must allow for visualization of signals on a map

## Design requirements:

Static data:

- must allow the front to represent the signals (choose an image)
- must allow the editor to configure the signals
- must allow the back to simulate the signals
- must be close to the industry model
- must allow for the modeling of composite signals, which represent several
  logical signals in a single physical signal

For the simulation:

- blocks must be generated for both the user and **pathfinding compatibility**
- each signal must know the **next compatible signal**
- each signal must know the **zones it protects**
- provide the **minimum necessary information** to the signaling modules for their operation
- be able to use a signaling module without having to instantiate a complete simulation
- be able to have modules that can be loaded in an independent order

```yaml
{
    # unique identifier for the signaling system
    "id": "BAL",
    "version": "1.0",
    # a list of roles the system assumes
    "roles": ["MA", "SPEED_LIMITS"],
    # the schema of the dynamic state of signals of this type
    "signal_state": [
        {"kind": "enum", "field_name": "aspect", values: ["VL", "A", "S", "C"]},
        {"kind": "flag", "field_name": "ralen30"},
        {"kind": "flag", "field_name": "ralen60"},
        {"kind": "flag", "field_name": "ralen_rappel"}
    ],
    # the schema of the settings signals of this type can read
    "signal_settings": [
        {"kind": "flag", "field_name": "Nf", "display_name": "Non-franchissable"},
        {"kind": "flag", "field_name": "has_ralen30", "default": false, "display_name": "Ralen 30"},
        {"kind": "flag", "field_name": "has_rappel30", "default": false, "display_name": "Rappel 30"},
        {"kind": "flag", "field_name": "has_ralen60", "default": false, "display_name": "Ralen 60"},
        {"kind": "flag", "field_name": "has_rappel60", "default": false, "display_name": "Rappel 60"}
    ],

    # these are C-like boolean expressions:
    # true, false, <flag>, <enum> == value, &&, || and ! can be used

    # used to evaluate whether a signal is a block boundary.
    "block_boundary_when": "true",

    # used for conflict detection. 
    "constraining_ma_when": "aspect != VL"
}
```

## Assumptions

- Each physical signal can be decomposed into a list of logical signals, all of which are associated with a signaling system.
- Blocks have a type
- It is possible to determine, given only the signal, its delimiting properties
- There are no sidings overlapping the end or beginning of a track
- Blocks not covered by tracks do not exist or can be ignored
- A train only uses one signaling system capable of transmitting the movement authority

# Design of Signaling Systems

Each signaling system has:

- A unique identifier (a string)
- A set of roles:
    - Transmission of Movement Authority
    - Transmission of speed limits
- Its signal state type, which must allow to:
    - Know if it is constraining due to reduced MA
    - Determine a graphical representation of the signal
    - Make a train react to this signal
- The signal parameters, used both in the front end to configure it and in the back end to determine the sidings information related to the signal
- The sidings condition, which determines if a signal delimits a siding

Note that if a signaling system has a dual role of transmitting MA and speed
limits, not all signals in this system are necessarily tasked with transmitting
speed limit information.

# Design of the blocks

The blocks have several attributes:

- a signaling system that corresponds to that displayed by its first signal.
- a **path**, which represents the block protected zones, and their expected state (in the same format as the road path)
- an **entry signal**, (optional when the block starts from a bumper)
- any **intermediate signals** if necessary (this is used with systems with distant signals)
- an **exit signal**, (optional when the block ends at a bumper)

The path is expressed from detector to detector in order to be able to match it with the road graph.

A few remarks:

- there may be several signaling systems superimposed on the same infrastructure. The model assumes that only one system at a time is active for train management
- a block does not have a state: one can rely on the dynamic state of the zones that make it up
- signals use blocks to know which zones are to be protected at a given time

# Design of the signals

Each physical signal consists of one or more logical signals, whose aspects are
combined to be represented on the field. During the simulation, logical signals
are treated as separate signals.

Each logical signal is associated with a signaling system, which defines if the
signal is transmitting Each logical signal can have a certain number of drivers.

## Unloaded signal

An **unloaded signal** corresponds to the object capable of transmitting
information to the train. Unloaded signals carry:

- a signaling system
- specific parameters for their signaling system (such as `Nf` and `has_ralen30` for bal)

Unloaded signals are used to statically describe the infrastructure and are
designed to be edited by the user.

Depending on the context, different mechanisms can be the source of this
information: they are called **drivers**. Each signal can carry several drivers.
For example, a BAL signal that is both a departure of the TVM block and a
departure of the BAL block will have two drivers: `bal-bal` and `bal-TVM`.

```yaml
{
    # signals must have location data.
    # this data is omited as its format is irrelevant to how signals behave

    "logical_signals": [
        {
            # the signaling system shown by the signal
            "signaling_system": "bal",
            # the settings for this signal, as defined in the signaling system manifest
            "settings": ["has_ralen30=true", "Nf=true"],
            # all the ways the signal can be driven
            # optional: if not present, we determine the drivers automatically
            "drivers": [
                # the signal can react to a following bal signal
                {"next_signaling_system": "bal"},
                # the signal can react to a following TVM signal
                {"next_signaling_system": "TVM"}
            ]
        }
    ]
}
```

```yaml
{
    # signals must have location data.
    # this data is omited as its format is irrelevant to how signals behave

    "logical_signals": [
        {
            # the signaling system shown by the signal
            "signaling_system": "bal",
            # the settings for this signal, as defined in the signaling system manifest
            "settings": ["has_ralen30=true", "Nf=true"],
            # all the ways the signal can be driven
            "drivers": [
                # the signal can react to a following bal signal
                {"next_signaling_system": "bal"}
            ]
        },
        {
            # the signaling system shown by the signal
            "signaling_system": "bapr",
            # all the ways the signal can be driven
            "drivers": [
                # the signal can react to a following bal signal
                {"next_signaling_system": "bapr"}
            ]
        }
    ]
}
```

A simplified string description of the signal type can be generated. Here: `bal[Nf=true,ralen30=true]+bapr`.

### Loading Signal Parameters

The first step of loading the signal is to characterize the signal in the signaling system.
This step produces an object that describes the signal.

During the loading of the signal:

   - the signaling system corresponding to the provided name is identified
   - the signal parameters are loaded and validated according to the signaling system spec
   - the signal's block role is evaluated from the expression

### Loading the Signal

Once the signal parameters are loaded, it becomes possible to load its drivers. For each driver:

   - the driver implementation is identified from the `(signaling_system, next_signaling_system)` pair
   - it is verified that the signaling system outgoing from the driver corresponds to the one of the signal
   - it is verified that there is no existing driver for the incoming signaling system of the driver

This step produces a `Map<SignalingSystem, SignalDriver>`, where the signaling
system is the one incoming to the signal.  It then becomes possible to construct
the loaded signal.

### Constructing Blocks

   - the framework creates blocks between signals following the routes present in the infrastructure, and the block properties of the signals
   - checks are made on the created block graph: it must always be possible to choose a block for each signal and each state of the infrastructure

### Speed Limits

Speed limits are represented as ranges on routes.
They start their life as ranges on track sections, and are lifted to ranges on routes as follows:

   - Directional speed limits by track are elevated to routes. The same speed limit can be on multiple routes.
   - For each speed limit, the route graph is traversed in reverse searching for signals capable of handling the limit:
       - Only speed limits not preceded by another limit with identical properties are considered
       - Each signal must signal its interest in a speed limit: Not concerned, Concerned but to be announced by another signal, or Concerned and terminal.
   - for each speed limit planned along the train's path, the signals in the announcement chain are added to the list of those to be simulated for the train.
    
```rust
enum SpeedLimitHandling {
    /** This signal isn't supposed to announce this limit */
    Ignore,
    /** This signal should announce this limit, but cannot */
    Error,
    /** This signal can announce this limit, and is part of an ongoing chain */
    Chain,
    /** This signal can announce this limit, and ends the chain */
    EndChain,
}

fn handles_speed_limit(
   self: SignalSettings,
   speed_limit: SpeedLimit,
   distance_mm: u64,
) -> SpeedLimitHandling;

fn handles_speed_limit_chain(
   self: SignalSettings,
   speed_limit: SpeedLimit,
   chain_signal: Signal,
   distance_mm: u64,
) -> SpeedLimitHandling;
```

### Block validation

The validation process helps to report invalid configurations in terms of signaling and blockage. The validation cases we want to support are:

- the signaling system may want to validate, knowing if the block starts / ends on a buffer:
    - the length of the block
    - the spacing between the block signals, first signal excluded
- each signal in the block may have specific information if it is a transition signal. Therefore, all signal drivers participate in the validation.

In practice, there are two separate mechanisms to address these two needs:

- the **signaling system module** is responsible for validating the signaling **within the block**
- the **signal drivers**, takes care of validating the transitions between blocks

```rust
extern fn report_warning(/* TODO */);
extern fn report_error(/* TODO */);

struct Block {
   startsAtBufferStop: bool,
   stopsAtBufferStop: bool,
   signalTypes: Vec<SignalingSystemId>,
   signalSettings: Vec<SignalSettings>,
   signalPositions: Vec<Distance>,
   length: Distance,
}

/// Runs in the signaling system module
fn check_block(
   block: Block,
);


/// Runs in the signal driver module
fn check_signal(
   signal: SignalSettings,
   block: Block, // The partial block downstream of the signal - no signal can see backward
);
```

### Signal lifecycle

Before a train startup:

- the path a of the train can be expressed as routes or as blocks. Those two paths should overlap.
- the signal queue a train will encounter is established.

During the simulation:
- along a train movement, the track ocuppation before it are synthetized
- when a train observes a signal, its state is evaluated

### Signal state evaluation

Signals are modeled as an evaluation function, taking a view of the world and returning the signal state

```kotlin

enum ZoneStatus {
   /** The zone is clear to be used by the train */
   CLEAR,
   /** The zone is occupied by another train, but otherwise clear to use */
   OCCUPIED,
   /** The zone is incompatible. There may be another train as well */
   INCOMPATIBLE,
}

interface MAView {
    /** Combined status of the zones protected by the current signal */
    val protectedZoneStatus: ZoneStatus
    val nextSignalState: SignalState
    val nextSignalSettings: SignalSettings
}

interface DirectSpeedLimit {
    /** Distance between the signal and the speed limit */
    val distance: Distance
    val speed: Speed
}

interface IndirectSpeedLimit {
    val distanceToNextSignal: Distance
    val nextSignalState: SignalState
    val nextSignalSettings: SignalSettings
}

interface SpeedLimitView {
    /** A list of speed limits directly downstream of the signal */
    val directSpeedLimits: List<DirectSpeedLimit>
    /** A list of speed limits which need to be announced in a signal chain */
    val indirectSpeedLimits: List<IndirectSpeedLimit>
}

fun signal(maView: MAView?, limitView: SpeedLimitView?): SignalState {
    // ...
}
```

The view should allow access to the following data:
 - a synthetized view of the zones downstream until the train MA
 - the block chain
 - the state of the signals downstream present in the train's block chain

### Signalization view path

The path of the signalization view is expressed in blocks:

- blocks can be added to extend the view
- the view can be reduced by removing blocks

### Simulation outside the train path

It is possible to simulate the signalization outside the train path:

   - if a signal gives way to blocks using different paths, it is simulated as if it were at the end of the itinerary and will therefore be at a standstill
   - if a signal gives way to blocks using the same path, it is simulated with other signals in the sequence, in a view built for this purpose

## Dependencies

For the block graph generation:

   - route graph. For each route:
       - `waypoints: List<DiDetector>`
       - `signals: OrderedMap<Position, UnloadedSignal>`
       - `speed_limits: RangeMap<Position, SpeedLimit>`, including the logic for train category limits
   - signalization systems
   - drivers

For evaluation:

   - train path in blocks
   - portion of the path to evaluate
   - drivers
   - state of the zones in the section to evaluate
    
## Operations

   - **Instantiating a view** creates a framework for observing signals
   - **Planning the path signals** to the view the blocks that the train will traverse
   - **Observing a signal** subscribe to the state of a signal (through the view)
   - **Passing a signal** signals that a signal has been passed by the train (through the view)

## Annexes

### Research Questions

   - Are there any blocks that overlap the end of a route? No (loic)
   - Are there any LL(2) signals? No in France
   - Are there signals that change behavior based on the active block in front of them? Yes, for slowdowns
   - Are there signals that are the start of blocks of different types (bal and bapr for example)? YES LOL, even tvm
   - Can the behavior of a signal depend on which block is active after the end of the current block? Yes, with slowdowns or blinking yellow
   - Do some signaling systems need additional information in the blocks? Kind of, there are slowdowns, but it's not specifically carried by the block
   - Is it nominal for a train to have multiple active signaling systems at the same time? No
   - When and by whom are the blocks generated?
   - What data is necessary for generating the blocks?

