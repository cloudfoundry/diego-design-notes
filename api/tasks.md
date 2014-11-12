# Tasks

Diego can run one-off work in the form of Tasks.  When a Task is submitted Diego allocates resources on a Cell, runs the Task, and then reports on the Task's results.  Tasks are guaranteed to run at most once.

When submitting a Task you must construct a valid `TaskCreateRequest`:

```json
{
    task_guid: "some-guid",
    root_fs: "",
    stack: "lucid64",
    ...
}
```

[back](README.md)