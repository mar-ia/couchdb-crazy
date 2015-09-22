# Sharding

##Scaling out

Normally you start small and grow over time. In the beginning you might do just fine with one node, but as your data
and number of clients grows, you need to scale out.

For simplicity we will start fresh and small. Turn off all the minions and wipe data to have a clean start. This will
remove all databases, but you are not storing anything in them anyway, are you?

Start minion1 and add a database to it. To keep it simple we will have 2 shards and no replicas.

    curl -X PUT "http://xxx.xxx.xxx.xxx:45984/small?n=1&q=2" --user daboss

If you look in data/shards you will find the 2 shards.

    data
        shards
            00000000-7fffffff
                small.1425202577.couch
            80000000-ffffffff
                small.1425202577.couch

Now, go to the admin panel

    http://xxx.xxx.xxx.xxx:45986/_utils

and look in the database `_dbs`, it is here that the metadata for each database is stored. As the database is
called small, there is a document called small there. Let us look in it. Yes, you can get it with curl too:

    curl -X GET "http://xxx.xxx.xxx.xxx:45986/_dbs/small"

    {
      "_id": "small",
      "_rev": "1-5e2d10c29c70d3869fb7a1fd3a827a64",
      "shard_suffix": [
        46,
        49,
        52,
        50,
        53,
        50,
        48,
        50,
        53,
        55,
        55
      ],
      "changelog": [
        [
          "add",
          "00000000-7fffffff",
          "minion1@xxx.xxx.xxx.xxx"
        ],
        [
          "add",
          "80000000-ffffffff",
          "minion1@xxx.xxx.xxx.xxx"
        ]
      ],
      "by_node": {
        "minion1@xxx.xxx.xxx.xxx": [
          "00000000-7fffffff",
          "80000000-ffffffff"
        ]
      },
      "by_range": {
        "00000000-7fffffff": [
          "minion1@xxx.xxx.xxx.xxx"
        ],
        "80000000-ffffffff": [
          "minion1@xxx.xxx.xxx.xxx"
        ]
      }
    }

* `"_id"`          The name of the database.
* `"_rev"`         The current revision of the metadata.
* `"shard_suffix"` The numbers after small and before .couch. The number of seconds after UNIX epoch that the database was created. Stored in ASCII.
* `"changelog"`    Self explaining.
* `"by_node"`      Which shards each node have.
* `"by_rage"`      On which nodes each shard is.

**Nothing here, nothing there, a shard in my sleeve **

Start minion2 and add it to the cluster. Check in `/_membership` that the minions talk with each other.

If you look in data on minion2, you will see that there is no directory called shards.

Go to Fauxton and edit the metadata for small, so it looks like this:

    {
      "_id": "small",
      "_rev": "1-5e2d10c29c70d3869fb7a1fd3a827a64",
      "shard_suffix": [
        46,
        49,
        52,
        50,
        53,
        50,
        48,
        50,
        53,
        55,
        55
      ],
      "changelog": [
        [
          "add",
          "00000000-7fffffff",
          "minion1@xxx.xxx.xxx.xxx"
        ],
        [
          "add",
          "80000000-ffffffff",
          "minion1@xxx.xxx.xxx.xxx"
        ],
        [
          "add",
          "00000000-7fffffff",
          "minion2@yyy.yyy.yyy.yyy"
        ],
        [
          "add",
          "80000000-ffffffff",
          "minion2@yyy.yyy.yyy.yyy"
        ]
      ],
      "by_node": {
        "minion1@xxx.xxx.xxx.xxx": [
          "00000000-7fffffff",
          "80000000-ffffffff"
        ],
        "minion2@yyy.yyy.yyy.yyy": [
          "00000000-7fffffff",
          "80000000-ffffffff"
        ]
      },
      "by_range": {
        "00000000-7fffffff": [
          "minion1@xxx.xxx.xxx.xxx",
          "minion2@yyy.yyy.yyy.yyy"
        ],
        "80000000-ffffffff": [
          "minion1@xxx.xxx.xxx.xxx",
          "minion2@yyy.yyy.yyy.yyy"
        ]
      }
    }

then press Save and marvel at the magic. The shards are now on minion2 too! We now have `n=2`!

If the shards are large, then you can copy them over manually and only have CouchDB syncing the changes from the
last minutes instead.

##Moving shards

When you get to `n=3` you should start moving the shards instead of adding more replicas. If you have `n<2` then
you have to stop all writes to the database or you will **LOSE DATA!** So increase n to `>1` before you are
starting to move shards around.

We will stop on `n=2` to keep things simple. Start minion number 3 and add it to the cluster. Then create the
directories for the shard

    mkdir -p data/shards/00000000-7fffffff

And copy over `small.1425202577.couch` that is in that directory. Do not move files between the shard directories
as that will make CouchDB very angry with you!

Edit the database document in `_dbs` again. Make it so that minion2 only have the shard `80000000-ffffffff`, and
minion3 only have shard `00000000-7fffffff`. Minion1 should not be touch this time.

First *"add"* then *"remove"* or is it *"delete"*? Is there a *"move"*? The changelog is nothing that CouchDB cares about.
So we can safely ignore it for now.

And behold, any documents added after we copied the shard was automatically synced from the replica on minion1.
Now we can safely delete the shard `00000000-7fffffff` from minion2.

Start minion4, add it the cluster and copy over shard `80000000-ffffffff` from minion1, update the database document,
let CouchDB sync and then delete the shard from minion1.

##Views

The views needs to be moved together with the shards. If you do not, then CouchBD have to rebuild them and this
will take time if you have a lot of data.

The views are stored in `.shards`

It is possible that it is recommended to let CouchDB rebuild the view every time you move a shard. As you have
more then one copy of every shard there is always a view that is up to date, while the new node is building the view.

##Reshard? No, Preshard!

Reshard? Nope. It can not be done. So do not create databases with to few shards.

If you can not scale out more because you set the number of shards to low, then you need to create a new cluster
and migrate over.

* 1:  Build a cluster with enough nodes to handle one copy of your data.
* 2:  Create a database with the same name, n=1 and with enough shards so you do not have to do this again.
* 3:  Set up 2 way replication between the 2 clusters.
* 4:  Let it sync.
* 5:  Tell clients to use both the clusters.
* 6:  Add some nodes to the new cluster and add them as replicas.
* 7:  Remove some nodes from the old cluster.
* 8:  Repeat 6 and 7 until you have enough nodes in the new cluster to have 3 replicas of every shard.
* 9:  Redirect all clients to the new cluster
* 10:  Turn off the 2 way replication between the clusters.
* 11: Shut down the old cluster and add the servers as new nodes to the new cluster.
* 12: Relax!

Creating more shards than you need and then move the shards around is called presharding. The number of shards you
need depends on how much data you are going to store. If you do not know, then the default of 8 is a good choice.
Creating to many shards increases the complexity without any real gain, so do not go crazy with shards.

There is a reason that the default is `q=8` ;)

##Final thoughts

You can have a different number of replicas for every shard. Why you would want that is beyond me, but it is possible.

Moving a shard is adding the shard to a node and removing it form another. Doing it in one step is easy, but not
the correct way. First add the new node as a new replica and let CouchDB sync. Then remove the shard from the old
node.

Better safe than sorry :)
