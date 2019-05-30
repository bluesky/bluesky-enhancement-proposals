# Bluesky Enhancement Proposal: RunEngine Server

## Abstract 
The basic idea is a process is created that contains a RunEngine. Clients in separate
processes could then request the execution of scans to be run on this "bluesky
server"

### Use Cases

#### GUIs
Currently GUI prototypes for bluesky store a "captive" RunEngine in the same
process as the user frontend. This will lead to potential instabilities as user
interactions and the actual run control processes have the potential to
interfere with one another. For instance, for a long running scan, you do want
a user closing a window to kill the background thread controlling hardware
abruptly. (We can argue about whether actually want the scan to stop as closing
a window will leave you without a clear way to manually pause or stop the scan,
but I think we can agree that we want to guarantee that  all objects are
unstaged and a stop document is produced. Keeping the RunEngine experimental
control and data publishing logic in a separate process we can greatly increase stability.

#### Centralization of RunEngine Configuration

While the ability to quickly stand-up a RunEngine and start executing a scan is
an incredibly powerful feature of `bluesky`, often instruments have a dedicated
set of callbacks that are expected to be present. Keeping track of this desired
configuration on where to save data e.t.c, and then ensuring that these are
kept in-sync between GUIs / IPython profiles / Jupyter Notebooks, is painful. A
benefit of going to a RunEngine server approach is you only need to launch the
server with the desired configuration, then each of these processes only needs
to know the information needed to know the location of the server.

#### Experimental Control

Centralizing the execution of scans can be beneficial to ensure that procedures
are not adversely affecting each other. The fact that the server will not run
plans in parallel would prevent operator errors where calibration routines are
accidentally requested during data collection for example. This could be
especially useful for automated tuning tasks that you want to run but do want
to interfere during another user requested operation. 

#### Experimental Gateway

EPICS does not lend itself well to security, that is the nature of a
distributed system. However, by setting up a RunEngine server that can be
reached by a user, you can allow a specific restricted set of operations on a
beamline without needing to give them permission to EPICS infrastructure at
all. This could be incredibly useful for beamlines that are operated remotely
via a web interface for example.

## Issues and Pull Requests

- https://github.com/bluesky/bluesky/pull/479

## Affected Repositories

- ophyd
- bluesky

## Open Design Decisions and Challenges

#### Device Proxies
Fortunately, `Msg` objects are trivial to serialize and ship over a wire, and
for the most part the simple string and number arguments inside are as well.
The one exception is how to correctly pass devices from client to server. The
objects with live EPICS connections have to live on the server, the client has
to be aware of which devices they can use in scans and how to specify each one.

- The server could load a fixed set of devices. A user could then simply use
  the "name" of devices instead of passing in the object itself. The server
  would then be responsible for keeping a registry for lookup. This would require
  names to be unique which is not currently enforced on any level.
- The server could use [happi](https://github.com/pcdshub/happi) to load devices on request.
- Some form of an `ophyd.DeviceProxy` could be created to exchange information
  with the server version of the device. This would mean objects on the client
  side could look very similar to the user as if they were actually making direct
  control layer calls. There is enough information in `Component` that I believe
  this is possible, but complex.

#### Local Callbacks
Currently, running callbacks in a separate process than the RunEngine is
possible. I am of the belief that the API would need to be improved around this
functionality in order to support this kind of running mode.

#### Communication Layer
What do we use to actually communicate with the server? ZMQ?

#### Messages vs. Plans
The server could either respond to individual messages, or take complete plans.
I see both as valid design decisions.

#### Synchronous Plan Execution 
There has been offline discussion regarding a RunEngine being able to have
multiple plans being run simultaneously. While I do believe that forcing users
to run one scan at a time is a feature, I also see it as a valid request that
multiple scans be run in parallel if the user acknowledges the risk.
