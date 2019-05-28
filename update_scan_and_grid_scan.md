# Bluesky Enhancement Proposal: ``scan`` and ``grid_scan`` update

## Abstract
The two main goals of this are:
    1. update the API surrounding snaking in ``grid_scan`` to match that used
    in the newly finished ``list_grid_scan`` and ``log_grid_scan`` (and their
    ``rel_*`` counterparts.
    2. bring ``scan`` and ``grid_scan`` functions in line with ``log_scan`` and
    ``log_grid_scan`` (as well as their `rel_*` counterparts). By this I mean
    making them wrappers around ``list_scan`` and ``list_grid_scan``.

## Affected Repositories
- bluesky

## Proposed Design
Below is the proposed layout of the new ``scan`` plan based on a call to
``list_scan``, note this is heavily based on the new ``log_scan`` plan in
bluesky PR#1194.

Take note that the plan metadata values that are shared with ``list_scan`` are
set in that function and therefore not applied here as well.

```python

def scan(detectors, *args, num=None, per_step=None, md=None):
    """
    Scan over one, or more, variable(s) in evenly-spaced steps simultaneously
    (inner product).

    Parameters
    ----------
    detectors : list
        list of 'readable' objects
    *args :
        For one dimension, ``motor, start, stop``.
        In general:

        .. code-block:: python

            motor0, start1, stop1,
            motor1, start2, start2,
            ...,
            motorN, startN, stopN

        Motors can be any 'settable' object (motor, temp controller, etc.)

        The values from above are defined as:

            motor : object
                any 'settable' object (motor, temp controller, etc.)
            start : float
                starting position for motor.
            stop : float
                ending position for motor.
    num : int
        number of steps
    per_step : callable, optional
        hook for customizing action of inner loop (messages per step)
        Expected signature: ``f(detectors, motor, step)``
    md : dict, optional
        metadata

    See Also
    --------
    :func:`bluesky.plans.rel_scan`
    :func:`bluesky.plans.grid_scan`
    :func:`bluesky.plans.rel_grid_scan`
    """

    if len(args) % 3 != 0:
        raise ValueError('The list of arguments handed to ``scan()`` must '
                         'contain multiples of the 3 arguments `motor`, '
                         '`start`, `stop`. However what was recieved is of '
                         'length {len(args)} which is not divisble by 3.'
                         'The full set of arguments are {}.'.format(len(args),
                                                                    args))

    list_args = []
    repr_args = []
    for motor, start, stop in partition(3, args):
        list_args.extend([motor, np.linspace(start, stop, num=num,
                                             endpoint=True)])
        repr_args.extend([repr(motor), start, stop])

    _md = {'plan_args': {'detectors': list(map(repr, detectors)),
                         'args': repr_args, 'num': num,
                         'per_step': repr(per_step)},
           'plan_name': 'scan'}
    _md.update(md or {})

    def inner_scan():
        return (yield from list_scan(detectors, *list_args,
                                     per_step=per_step, md=_md))

    return (yield from inner_scan())
```

## What are the benefits
The main benefits I see are:
    1. It reduces the complexity of the code in ``bluesky.plans`` and
    ``bluesky.plan_patterns`` (in the later removing the need for the
    ``outer_product`` and ``inner_product`` plan patterns).
    2. It reduces the amount of duplicate code we have regarding plans, by
    allowing common lines to be contained in `list_scan`.
    3. Future bug fixes to one type of plan will permeate to others, as they
    are linked by ``list_scan``.
    4. It will by default introduce the new 'snaking API' to ``grid_scan``
    (although we will need to support the old version for back-compatibility)

## Posible extensions
In principle this could also be extended to the other non-adaptive scans (like
``spiral``, ``fermat_spiral``, ``square_spiral``, etc). I am less
convinced about the benefits of this, as it would require rewriting the
`plan_patterns` for these to return ``motor, pos_list`` argument pairs instead
of cyclers.
