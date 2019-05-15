# Bluesky Enhancement Proposal: Telemetry

## Abstract

The goal is to capture the timing of every asynchronous action—meaning that the action that is done through a Status object—along with the context necessary to interpret that action, such as initial and final position of a movement or acquire time for an exposure. The set of asynchronous actions currently includes trigger, set, kickoff, complete. It should soon include stage and unstage as well when they operate via Status objects (though that work is not in scope for this proposal).

This will enable future work on *time estimation* for bluesky plans. It is a necessary but not sufficient step toward that feature. Time estimation will additionally need to account for the time spent doing many fast operations, such as read() and describe(). It was proposed to record telemetry for these as well, but that proposal was rejected because recording the telemetry takes time comparable to the time of the action itself.

## Affected Repositories

- ophyd

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
        telemetry_components = ()
    
        def _finished(self):
            configuration = {}
            for component_name in telemetry_components:
                configuration[component_name] = getattr(self.device, component_name).get()
            dt = time.time() - self.start_time
            record_telemetry({'dt': dt, 'configuration': configuration})
    
    class MoveStatus(DeviceStatus):
        ...
    
        def _finished(self):
            ...
            record_telemetry({'dt': dt, 'configuration': configuration, 'action': {'initial': self.initial, 'target': self.target}})
    ```

In order to capture the context necessary to interpret a given timing (velocity, exposure time, etc.) we need to capture configuration that will be specific to a given device. Therefore we will add `EpicsMotorStatus` and `ADStatus` that get the signals that they need.

    ```python
    # New, Device-specific flavors of Status object
    
    class ADStatus:
        telemetry_compoments = ('cam.acquire_time',)
    
    class EpicsMotorStatus:
        telemetry_components = ('velocity',)
    ```

## Rejected Alternatives

- Hard-code the `telemetry_components` in `set`, `trigger`, etc. Rejected because we will want to expose that list of components somewhere public for future work on time estimation in bluesky
- Hard-code the `telemetry_components` in device-specific Status objects. Rejected for the same reason.
- Add a `telemetry` attribute, used similarly to `kind`, but set once at Component initialization time and not mutable. Rejected because the list of telemetry components depends on the kind of action (different for `set` vs `trigger`) so it can’t be generically set on a Component.
- Make `'telemetry'` another `kind`. Rejected for the same reason as above, and also because it would make `kind` more complex.
- Add a separate `ETA` or `Telemetry` object to store this information, passed into `OphydObj`. This is the approach that we decided on in the initial pass at this feature, but it’s not needed for *telemetry* specifically (maybe later for time estimation) and can be added later. Thus, rejected as out of scope for this proposal.

## Open Questions
- Exact structure of telemetry payload

## Links to documents from prior rounds of discussion on this feature
- https://www.dropbox.com/s/ch7j1x6j4njby7e/ETA%20overview.docx?dl=0
- [+Outline of ETA co-routine.](https://paper.dropbox.com/doc/Outline-of-ETA-co-routine.-R9emFtWxa0bYjXUYzywno) 
- [+Outline of changes for plan estimation](https://paper.dropbox.com/doc/Outline-of-changes-for-plan-estimation-rwFPrkARP6c2iTv8voHdl) 
