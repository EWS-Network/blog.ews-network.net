.. link:
.. description: DNS - Bind9 / Named
.. tags: Linux, Bind9, Named
.. date: 2013/11/03 14:21:11
.. title: DNS - Bind9/Named [Part 5]
.. slug: dns-bind9named-part-5

Forwarding is also one of the very interesting capacity of bind. Imagine, we have somewhere.net hosted by primary DNS ns1 and we have a really big zone "tutorials" which is held by a secondary DNS ns2. We'd like that when we query "test.tutorials.somewhere.net" on master server (which does not have the zone hosted in his configuration files), the ns1 server will ask to ns2.somewhere.net. There, ns0 will act as a forwarder.

To do so, what we're going to do is tell the ns0 server that "tutorials" is held by ns2. So in our db.somewhere.net we have :

.. code::

   ; db for somewhere.net
   ;
   $TTL    86400
   @		IN    SOA	ns1.somewhere.net. root.somewhere.net. (
				1         ; Serial
   				604800         ; Refresh
   				86400         ; Retry
   				2419200         ; Expire
   				86400 )       ; Negative Cache TTL
   ;
   ; Here we define the nameservers of the domain.
   @		IN    NS	ns1.somewhere.net.
   @		IN    NS	ns2.somewhere.net.
   ;
   ;Here we set the MX records for our domain
   @		IN    MX    10  smtp.somewhere.net.
   ;
   ; Now we set the IP of the nameservers - Use yours
   ns1		IN    A		192.168.1.1
   ns2		IN    A		172.16.1.1
   ;
   ; Now, we set some zones
   www		IN    A		192.168.1.10
   smtp		IN    A		192.168.1.2

Great. We also have to add this zone as a forwarded one in our named.conf.local

.. code::

   zone "tutorials.somewhere.net"
   {
	type forward;
	forward only;
	forwarders { 172.16.1.1; };
   };
