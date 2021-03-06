# wtf-json File Format

This format is a non-optimized, human-readable event stream. It is designed to
be much easier to generate than the binary format at the cost of file size
and load time. Prefer to use the binary format where possible.

## Structure

A complete file contains a list of objects, with each object having a certain
type. In JSON, this looks like:

    [
      {
        ...
      },
      {
        ...
      }
    ]

In order to support generators that cannot properly close the enclosing array,
parsers of the format also support omitting the trailing `]` and the presence of
trailing commas:

    [
      {
        ...
      },
      {
        ...
      },   <-- trailing comma
           <-- no trailing ]

Optionally the array can be wrapped inside of an object:

    {
      'events': [...]
    }

## Entries

### Header

Each trace should include as its first entry a header object. This header
describes the trace file event format and adds additional information to help
merge trace files together. If the header is omitted the default values below
are used.

    {
      "type": "wtf.json#header",
      "format_version": 2,
      "high_resolution_times": true,
      "timebase": 0
    }

* `format_version`: major version of the format, must be 2.
* `high_resolution_times`: whether the times in the file are high resolution.
This indicates that the times have higher than millisecond precision.
* `timebase`: unix time since the epoch that is added to all time values in the
trace. A timebase of 1000 means that a `time` of 5 would be seen as 1005.
* `time_offset`: milliseconds to offset the time from any previously loaded
files. This overrides any timebase setting and should only be used in 'additive'
files.

### Event Definitions

Any event that will be seen in the stream must be defined before it is seen.
Events are defined by a signature that can be either a simple name or a
function-like signature with arguments. Events are either `scopes` or
`instance` types, and can contain optional flags. Events can be defined anywhere
in the stream but must be defined before use.

To decrease filesize it's possible to assign an `event_id` that will be
referenced instead of the event name. This is optional.

    {
      "type": "wtf.event#define",
      "signature": "my.custom#event(uint32 foo)"
      "class": "scope",
      "flags": 0,
      "event_id": 0
    }

The minimal possible definition for a scope event:

    {
      "type": "wtf.event#define",
      "signature": "my.custom#event(uint32 foo)"
    }

* `signature`: a function-like signature. See [api](api.md) for more
information. The event name here can be referenced by `event` in event objects.
* `class`: either `scope` (the default) or `instance`.
* `flags`: optional flags values. Currently unused.
* `event_id`: optional file-unique number to assign the event. This can be used
instead of the name in `event`.

### Events

Event objects are the majority of a file indicating the occurrence of an event,
its time, and optionally any arguments.

    {
      "event": 0,
      "time": 1234,
      "args": [
        4567
      ]
    }

`event`: either the event name from the definition `signature` or the `event_id`
if it was assigned. This is a reference to a previously defined event.
`time`: the time the event occurred, relative to the `timebase`.
`args`: an optional list of arguments for the event, if any were indicated in
the `signature`.

### Context Info

TODO(benvanik): context info

### Zones

TODO(benvanik): support zones with some sugar

      {
        "event": "wtf.zone#create",
        "time": 0,
        "args": [
          1000,         <-- ID
          "zone name",
          "script",
          "http://some/url.js"
        ]
      },
      {
        "event": "wtf.zone#set",
        "time": 0,
        "args": [
          1000          <-- ID
        ]
      }

### Flows

TODO(benvanik): support flows

## Builtin Events

To help reduce the file length, certain system events are built in to the
format.

### wtf.scope#leave

Leaves a scope event. This is implicitly defined as:

    {
      "type": "wtf.event#define",
      "signature": "wtf.scope#leave",
      "event_id": -1
    }

This allows for the shorthand for leaving an entered scope:

    {
      "event": -1,
      "time": 1234
    }

## Example

Smallest possible file:

    [
      {
        "type": "wtf.event#define",
        "signature": "my.custom#event",
        "class": "instance"
      },
      {
        "event": "my.custom#event",
        "time": 123450001
      },
      {
        "event": "my.custom#event",
        "time": 123450002
      }
    ]

Efficient file:

    [
      {
        "type": "wtf.json#header",
        "format_version": 2,
        "timebase": 123450000
      },
      {
        "type": "wtf.event#define",
        "signature": "my.custom#scopeEvent",
        "event_id": 1
      },
      {
        "type": "wtf.event#define",
        "signature": "my.custom#instanceEvent",
        "event_id": 2
      },
      {
        "event": 1,     <-- enter #scopeEvent
        "time": 1
      },
      {
        "event": 2,     <-- #instanceEvent
        "time": 2
      },
      {
        "event": -1,     <-- leave #scopeEvent
        "time": 3
      }
    ]
