#Setup

##Who?

If you run away screaming when someone say "Erlang cluster", then you should wait for the final release.

##Why?

You want to take CouchDB2.0 for a ride and play(keyword!) production. As dev/run starts 3 nodes on one server and that is not how you usually do it in production, you need something else than developer preview.

##How?

Some very minor hacking :)

#Requirements?

* NodeJS with npm (any version should work, I use the old 0.10.30)
* Rebar 2.3.1     (or newer)
* Erlang          (the later the better, I use 17.4)
* Iptables        (Expose the clusterports to the Internet and you are FUCKED!)
* git             (any version)
* Computers       (Yes, plural, we are building a cluster here!)

## Firewall

If you do not have a firewall between your servers, then you can skip this.

CouchDB2.0 uses the port 5984 just as 1.x.y, but is also uses 5986 for the admin interface. To keep out of the way we will use the ports 45984 and 45986.

Erlang uses TCP port 4369 (EPMD) to find nodes, so all servers must be able to speak to each other on this port.
In an Erlang Cluster, all nodes are connected to all other nodes. A mesh.

Every Erlang application then uses other ports for talking. Random ports! Yay \ö/ 

No, this needs to be fixed. Let us settle for the range TCP 9100-9200. Open up those ports in your firewalls and it is time to test it.

You need 2 servers with working hostnames. I will call them server1 and server2. The `.` is to Erlang what `;` is to C.

Then on server1:

    erl -sname bus -setcookie 'brumbrum' -kernel inet_dist_listen_min 9100 -kernel inet_dist_listen_max 9200

On server2 run:

    erl -sname car -setcookie 'brumbrum' -kernel inet_dist_listen_min 9100 -kernel inet_dist_listen_max 9200

This gives us 2 Erlang shells. shell1 on server1, shell2 on server2.

In shell1:

    net_kernel:connect_node(car@server2).

If that returns true, then you have a erlang cluster, and the firewalls are
open. If you get false or nothing at all, then you have a problem with the
firewall.

## First time in Erlang. Time to play!

Run in both shells:

    register(shell, self()).

shell1:

    {shell, car@server2} ! {hello, from, self()}.

shell2:

    flush().
    {shell, bus@server1} ! {"It speaks!", from, self()}.

shell1:

    flush().

To close the shells, run in both:

    q().
