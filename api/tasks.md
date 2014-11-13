# Tasks

Diego can run one-off work in the form of Tasks.  When a Task is submitted Diego allocates resources on a Cell, runs the Task, and then reports on the Task's results.  Tasks are guaranteed to run at most once.

When submitting a Task you must construct a valid `TaskCreateRequest`:

```json
{
    task_guid: "some-guid",
    domain: "some-domain",
    stack: "lucid64",

    root_fs: "docker:///docker-org/docker-image",
    cpu_weight:
    disk_mb:
    memory_mb:
    env: 

    action:  ACTION,

    result_file:
    completion_callback_url: "http://optional/callback/url",

    log_guid: ,
    log_source: ,

    annotation: "arbitrary-metadata"
}
```

<dl>
<dt>`task_guid`</dt>
<dt>
    The task guid...
</dt>
</dl>

[back](README.md)