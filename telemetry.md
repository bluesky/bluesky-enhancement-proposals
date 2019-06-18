# Bluesky Enhancement Proposal: Telemetry

## Abstract

The goal is to capture the timing of every asynchronous action—meaning that the action that is done through a Status object—along with the context necessary to interpret that action, such as initial and final position of a movement or acquire time for an exposure. The set of asynchronous actions currently includes trigger, set, kickoff, complete. It should soon include stage and unstage as well when they operate via Status objects (though that work is not in scope for this proposal).

This will enable future work on *time estimation* for bluesky plans. It is a necessary but not sufficient step toward that feature. Time estimation will additionally need to account for the time spent doing many fast operations, such as read() and describe(). It was proposed to record telemetry for these as well, but that proposal was rejected because recording the telemetry takes time comparable to the time of the action itself.

## Affected Repositories

- ophyd

## Timing Sequence

Here are the actions that require network activity:

1. Request is sent to begin action (moving setpoint, starting acquisition, ...).
2. Response is received indicating that action has completed.
3. Additional readings are taken of Device Components needed to interpret
   telemetry (velocity, exposure time, ...).
4. The payload of telemetry information is inserted into one or more database.

When in this sequence should the "clock" start and stop?

From the point of view of profiling hardware performance of time, we wish to
isolate 1--2. Thus, if we later expand the amount of extra information collected
in (3) it will not distort our measurement of the hardware's action itself.

But from the point of view of producing accurate time estimations in bluesky,
the duration of the entire procedure is of interest. Of course we cannot measure
(4) because it is only known after the payload has been inserted. But we might
be interested in including (3) if in practice it can be used to significantly
improve accuracy of time estimations.

We propose to include three times:

* ``start_time`` --- UNIX epoch time taken before (1)
* ``command_dt`` --- difference in seconds between a time taken after (2) and
  the ``start_time``
* ``telemetry_overhead_dt`` --- difference in seconds between a time taken after (3) and the
  time taken after (2)

## Proposed payload for database
payload = {'device_name': 'name of device',
           'command': 'name of command ('set', 'trigger', ...),
           'start_time': 'time the command was started',
           'command_dt': 'time taken to complete the command',
           'telemetry_overhead_dt': 'time taken to read device components and status attributes to be recorded',
           'device_components':{'component_name': value for each component in status.device_components},
           'status_attributes':{'attribute': value for each attribute in status.status_attributes}}

## Proposed Design

Key aspects

- Global function for adding telemetry destinations (can be 0, 1, or N) — could be local file, MongoDB, Elastic, etc.
- Global function for inserting a new telemetry payload
- The telemetry payloads are written and submitted from inside Status objects
- No changes needed to OphydObj

```python
# New module-level functions in ophyd

telemetry_destinations = []

def register_telemetry(func):
    telemetry_destinations.append(func)

def record_telemetry(info):
    for destination in telemetry_destinations:
        destination(info)

#  Update existing status objects

class DeviceStatus:
    ...
    def __init__(self, device, **kwargs, command=None):
        self.device = device
        self._watchers = []
        self.command = command 
        # if command is None no telemetry is stored, else it must be
        # 'set' or 'trigger' or ....
        super().__init__(**kwargs)
    
    device_components = ()
    status_attributes = ()

    ...
  
    def _finished(self):
        if self.command:  # For backwards compatibility
            command_time = time.time()
            payload = {'device_name': self.device.name,
                       'command': self.command, 
                       'device_components':{},
                       'status_attributes':{}}

            for component_name in self.device_components:
                payload['device_components'][component_name] = getattr(self.device, component_name).get()
            for attribute_name in self.status_attributes:
                payload['status_attributes'][attribute_name] = getattr(self, attribute_name).get()
            payload['command_dt'] = command_time-self.start_time
            payload['telemetry_overhead_dt'] = time.time()-command_time
            record_telemetry(payload)

class MoveStatus(DeviceStatus):
    ...
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs, command='set')
    
    device_components = ('settle_time')
    status_attributes = ( 'start_pos', 'finish_pos')
    ...
```

In order to capture the context necessary to interpret a given timing (velocity, exposure time, etc.) we need to capture configuration that will be specific to a given device. Therefore we will add `EpicsMotorStatus` and `ADStatus` that get the signals that they need.

```python
# New, Device-specific flavors of Status object

class ADTriggerStatus(DeviceStatus):
    ...
    device_components = ('cam.acquire_time', 'cam.acquire_period', 'cam.num_images', 'cam.trigger_mode', 'settle_time')
    status_attributes = ()
    ...

class EpicsMotorSetStatus(Move_status):
    ...
    device_components = ('settle_time', 'velocity')
    status_attributes = ('start_pos', 'finish_pos')
    ...
```

## Rejected Alternatives

- Hard-code the `telemetry_components` in `set`, `trigger`, etc. Rejected because we will want to expose that list of components somewhere public for future work on time estimation in bluesky
- Hard-code the `telemetry_components` in device-specific Status objects. Rejected for the same reason.
- Add a `telemetry` attribute, used similarly to `kind`, but set once at Component initialization time and not mutable. Rejected because the list of telemetry components depends on the kind of action (different for `set` vs `trigger`) so it can’t be generically set on a Component.
- Make `'telemetry'` another `kind`. Rejected for the same reason as above, and also because it would make `kind` more complex.
- Add a separate `ETA` or `Telemetry` object to store this information, passed into `OphydObj`. This is the approach that we decided on in the initial pass at this feature, but it’s not needed for *telemetry* specifically (maybe later for time estimation) and can be added later. Thus, rejected as out of scope for this proposal.

## Links to documents from prior rounds of discussion on this feature
- https://www.dropbox.com/s/ch7j1x6j4njby7e/ETA%20overview.docx?dl=0
- [+Outline of ETA co-routine.](https://paper.dropbox.com/doc/Outline-of-ETA-co-routine.-R9emFtWxa0bYjXUYzywno) 
- [+Outline of changes for plan estimation](https://paper.dropbox.com/doc/Outline-of-changes-for-plan-estimation-rwFPrkARP6c2iTv8voHdl) 
