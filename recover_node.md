Recover MongoDB node that is out of sync
========================================

1. Shut down the secondary MongoDB node.
1. Replace the node's `local`, `journal` and `wildcat` folders in `/mnt/mongodb/data` with copies from a healthy server.
1. Start MongoDB on the node. It will automatically sync itself up to date with the primary node.

