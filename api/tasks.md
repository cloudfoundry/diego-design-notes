# Tasks

Diego can run one-off work in the form of Tasks.  When a Task is submitted Diego allocates resources on a Cell, runs the Task, and then reports on the Task's results.  Tasks are guaranteed to run at most once.

## Describing Tasks

When submitting a Task you must construct a valid `TaskCreateRequest`:

```
{
    "task_guid": "some-guid",
    "domain": "some-domain",

    "stack": "lucid64",

    "root_fs": "docker:///docker-org/docker-image",
    "env": [
        {"name": "ENV_NAME_A", "value": "ENV_VALUE_A"},
        {"name": "ENV_NAME_B", "value": "ENV_VALUE_B"}
    ],

    "cpu_weight": 57,
    "disk_mb": 1024,
    "memory_mb": 128,

    "action":  ACTION (see below),

    "result_file": "/path/to/return",
    "completion_callback_url": "http://optional/callback/url",

    "log_guid": "some-log-guid",
    "log_source": "some-log-source",

    "annotation": "arbitrary metadata"
}
```

Let's describe each of these fields in turn.

#### Task Identifiers

##### `task_guid`
##### `domain`

#### Task Placement

##### `stack`

#### Container Contents and Environment

##### `root_fs`
##### `env`

#### Container Limits

##### `cpu_weight`
##### `disk_mb`
##### `memory_mb`

#### Actions

#### Task Completion and Output

##### `result_file`
##### `completion_callback_url`

#### Logging

##### `log_guid`
##### `log_source`

#### Attaching Arbitrary Metadata

##### `annotation`

## Retreiving Tasks

To learn that a Task is completed you must either register a `completion_callback_url` or periodically poll the API to fetch the Task in question.  In both cases, you will receive an object **that includes all the fields on the `TaskCreateRequest`** and the following additional fields:

```
{
    ... all TaskCreateRequest fields...

    "created_at": UNIX_TIMESTAMP_IN_NANOSECONDS,
    "failed": true/false,
    "failure_reason": "why it failed",
    "result": "the contents of result_file",
    "state": "PENDING", "CLAIMED", "RUNNING", "COMPLETED", or "RESOLVING"
}
```

Let's describe each of these fields in turn.

##### `created_at`

##### `failed`

##### `failure_reason`

##### `result`

##### `state`

## The Task lifecycle



[back](README.md)