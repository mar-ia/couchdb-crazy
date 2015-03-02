#Deploy

##Spawning time

Time to make a minon!

    cd ..
    cp -r minion_skeleton minion1
    cd minion1
    mv start_minion.sh start_minion1.sh

Open `start_minion.sh` and `etc/local.ini` and change all

    minion_skeleton

to

    minion1

In `etc/local.ini` change the uuid from "change_me_later" to an actual uuid. Ask a running couchdb `/_uuids` for one.
And do not forget to change `xxx.xxx.xxx.xxx` to an ip.

Open etc/vm.args and change

    {{node_name}}

to

    -name minion1@xxx.xxx.xxx.xxx

where `xxx.xxx.xxx.xxx` is the IP of the server you are going to run it on.

If you are going to run more than one minion per server, you have to change the ports(45984,45986) as well.

Repeat until you have all the minions you want.

##Deploy

You can now move the minions to their servers. If your servers run different versions, distributions and what not, you
probably know how to deal with it :)

##Boot

Run `./start_minionX.sh` on every server to start the cluster.

##Connect Cluster

Go to `http://server1:45984/_membership` to see the name of the node and all the
nodes it knows about and are connected too.

    curl -X PUT "http://xxx.xxx.xxx.xxx:45984/_membership" --user daboss
    {"all_nodes":["minion1@xxx.xxx.xxx.xxx"],"cluster_nodes":["minion1@xxx.xxx.xxx.xxx"]}

* "all_nodes" are all the nodes thats this node knows about.
* "cluster_nodes" are the nodes that are connected to this node.

Time to add a node:

    curl -X PUT "http://xxx.xxx.xxx.xxx:45986/_nodes/minion2@yyy.yyy.yyy.yyy" -d {}

Now look at `http://server1:45984/_membership` again.

    {"all_nodes":["minion1@xxx.xxx.xxx.xxx","minion2@yyy.yyy.yyy.yyy"],"cluster_nodes":["minion1@xxx.xxx.xxx.xxx","minion2@yyy.yyy.yyy.yyy"]}

A 2 node cluster! \ö/

`http://yyy.yyy.yyy.yyy:45984/_membership` will show the same thing, so you only have to add a node once.

Repeat until all nodes are connected.

##Shutdown

Ctrl+C in the shell you ran `./start_minionX.sh` to shut down a node.

##Start

Run `./start_minionX.sh` to start a node.


