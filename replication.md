Replication
===========

### How to determine the oldest entry in the oplog

    use local
    new Date(db.oplog.rs.find().sort({$natural:1}).limit(1).next()["ts"]["t"])

    # Number of hours from now until the last entry in the oplog
    (new Date() - new Date(db.oplog.rs.find().sort({$natural:1}).limit(1).next()["ts"]["t"])) / (60 * 60 * 1000)
