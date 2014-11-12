# Diego API Docs

**The Diego API is still work-in-progress and subject to change.**

Here's an outline:

- [API Overview](overview.md)
- [Understanding Tasks](tasks.md)
- [Understanding Long Running Processes (LRPs)](lrps.md)
- API Reference
    - Tasks
        - [Creating Tasks](create_tasks.md)
        - [Fetching Tasks](fetch_tasks.md)
        - [Cancelling Tasks](cancel_tasks.md)
        - [Deleting Tasks](delete_tasks.md)
    - LRPs (desired state)
        - [Desiring LRPs](desire_lrps.md)
        - [Fetching DesiredLRPs](fetch_desired_lrps.md)
        - [Updaing DesiredLRPs](update_desired_lrps.md)
        - [Deleting DesiredLRPs](delete_desired_lrps.md)
    - LRPs (actual state)
        - [Fetching ActualLRPs](fetch_actual_lrps.md)
        - [Restarting Individual ActualLRPs](restart_actual_lrps.md)
    - [Listing Cells](list_cells.md)
    - [Maintaining Freshness](maintain_freshness.md)