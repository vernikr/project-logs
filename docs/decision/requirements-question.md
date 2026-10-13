# One decision before architecture selection

The first architecture choice is not the database or language. It is what the system must guarantee when
the workstation and server are not simultaneously available.

## Question

**Which guarantee should be mandatory for version 1? Choose one primary priority:**

1. **Remote agent access first.** Logs from capable projects should reach the server and be queryable by
   cloud agents with minimal delay. Local-only and restricted projects may use an export queue.
2. **Local ownership first.** Logs remain local by default and are sent to the server only by explicit,
   controlled export. Remote access is useful but secondary to data locality and offline operation.
3. **Minimal integration first.** Adding logging to a project should take almost no project-specific work;
   the server view may be incomplete or delayed until stronger adapters exist.

This choice determines whether the first design should be push/queue-centric, local-first with replication,
or adapter/collector-centric. It also sets the acceptable failure mode when the remote path is unavailable.

Reply with `1`, `2`, or `3`, plus any hard requirement that changes the choice (for example: retention,
private-network-only access, or a ban on sending payload text).
