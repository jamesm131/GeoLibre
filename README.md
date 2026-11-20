# GeoLibre PR queue

Branches waiting for a free PR slot on opengeos/GeoLibre. GitHub caps how many PRs this account can have open there, so the next one goes up only when one merges or closes.

- `queue.json`: open PRs and queued branches, in the order they should be opened. Each queued entry names its head branch on this fork, its title, and its description in `bodies/`.
- `state.json`: what the daily check has already handled (`lastRun`, and per PR the review/comment ids already dispatched).

The daily T3 scheduled task reads and updates these files. To add a branch, push it to this fork and append an entry to `queued` with its body file.
