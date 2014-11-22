# Cells

To fetch all available Cells:

```
GET /v1/cells
```

This will return an array of `CellResponse` objects.  A `CellResponse` is of the form:

```
{
    cell_id: "some-cell-id",
    stack: "stack"
}
```

[back](README.md)


