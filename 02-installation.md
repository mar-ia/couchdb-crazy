#Installation

##Git

Create  a directory to work in. If you put in another place, change accordingly.

    mkdir /home/username/couchdb2.0
    cd /home/username/couchdb2.0
    git clone https://github.com/apache/couchdb.git
    cd couchdb
    ./configure
    make

And we have the latest master. We will leave this directory alone so we easy can upgrade when there is a new version.

##The Skeleton of a minion

    cd ..
    cp -r couchdb minion_skeleton
    cd minion_skeleton

This gives us a skeleton to have as the base for all our nodes. As the developer preview uses node1, node2 and node3,
we will use minions instead to avoid conflict.

For some reason boot_node.erl is not compiled during make, and is in dev. Let us move it to the root just for the
sake of it.

    cp dev/boot_node.erl .
    rm dev -rf
    erlc boot_node.erl

We need a place to put the databases, data will do.

    mkdir data

We need to force CouchDB to use the ports we have opened in the firewall. If you do not have a firewall, you can skip
this step. Open rel/files/sys.config and make the lines 28-32 look like this:

            ]},
            {inet_dist_listen_min, 9100},
            {inet_dist_listen_max, 9200}
        ]}
    ].

The configuration files are in rel/overlay/etc we copy them so we can get to them easier.

    cp -r rel/overlay/etc .

Edit etc/vm.args to set the cookie for the cluster. Change

    -setcookie monster

to

    -setcookie eviloverlord

So that we in no way are in conflict with the developer preview.

Open etc/local.ini and go to the bottom and under `[admin]` add an admin from
your 1.6.1 install. No admin party here.

    daboss = -pbkdf2-2837642876498764294762cabab etc etc etc

Now we are going to use `local.ini` to overwrite all `{{variable_name}}` in `default.ini`. We will use port 45984 and
45986 as they are not used by the developer preview. Remember that the uuid must be the same on all the nodes in a
cluster.

    [vendor]
    name = The Apache Software Foundation

    [couchdb]
    database_dir = /home/username/couchdb2.0/minion_skeleton/data
    view_index_dir = /home/username/couchdb2.0/minion_skeleton/data
    uuid = must_be_the_same_on_all_the_nodes

    [chttpd]
    port = 45984
    bind_address = xxx.xxx.xxx.xxx
    docroot = /home/username/couchdb2.0/minion_skeleton/share/www

    [httpd]
    port = 45986
    bind_address = xxx.xxx.xxx.xxx

    [query_servers]
    javascript = /home/username/couchdb2.0/minion_skeleton/bin/couchjs /home/username/couchdb2.0/minion_skeleton/share/server/main.js
    coffeescript = /home/username/couchdb2.0/minion_skeleton/bin/couchjs /home/username/couchdb2.0/minion_skeleton/share/server/main-coffee.js

    [httpd_global_handlers]
    favicon.ico = {couch_httpd_misc_handlers, handle_favicon_req, "/home/username/couchdb2.0/minion_skeleton/share/www"}
    _utils = {couch_httpd_misc_handlers, handle_utils_dir_req, "/home/username/couchdb2.0/minion_skeleton/share/www"}

##Start

We need a bash script to start a minion, let us call it `start_minion.sh` put it in the root:

    #!/bin/bash

    erl -args_file /home/username/couchdb2.0/minion_skeleton/etc/vm.args -config /home/username/couchdb2.0/minion_skeleton/rel/files/sys.config -couch_ini /home/username/couchdb2.0/minion_skeleton/etc/default.ini /home/username/couchdb2.0/minion_skeleton/etc/local.ini -reltool_config /home/username/couchdb2.0/minion_skeleton/rel/reltool.config -parent_pid $$ -pa /home/username/couchdb2.0/minion_skeleton/src/* -pa /home/username/couchdb2.0/minion_skeleton/src/*/* -s boot_node

Then make it executable:

    chmod +x start_minion.sh
