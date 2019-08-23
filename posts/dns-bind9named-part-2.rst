.. link:
.. description: DNS - Bind9 / Named
.. tags: Linux, DNS, Bind9, Named
.. date: 2013/11/03 14:01:27
.. title: DNS - Bind9/Named [Part 2]
.. slug: dns-bind9named-part-2

Now we have a DNS Server working correctly which serves our domain. We also can want to restrict access to this server. What we're going to see :

- Restricting access
- Serve domains according to their original network

So as for anything, we will talk about ACLs. That's very easy to add an ACL in bind. Just edit a file like "lan.acl" and in it you can create your acl rules :

.. code::

   # ACL for VMs on the cloud
   acl "cloud_vms" { 10.0.0.0/24; };
   # ACL for admin lans
   acl "admin_lans" { 192.168.42.0/54; 192.168.150.0/27; };

Now thanks to this configuration, we can identify the origin of the clients. We also have to add the config file in named.conf like this :

.. code::

   include "/etc/bind/lan.acl";

So now, as we've seen just before, we allowed "any" to make queries. So, if you want to limit to the clients only, just edit named.conf.options

.. code::

   allow-query { cloud_vms; admin_lans; localhost; };

But this is very limited. You will serve the same domains to all your clients. And what we want is to serve the same domain with different IPs, we can use the "view" instruction. Just like this :

.. code::

   view "Cloud"
   {
	match-clients { cloud_vms; };
	zone "somewhere.net"
	{
		type master;
		file "/etc/bind/db.somewhere.net.cloud";
	};
   };
   view "Admin"
   {
	match-clients { admin_lans; };
	zone "somewhere.net"
	{
		type master;
		file "/etc/bind/db.somewhere.net.admin";
	};
   };

If you get an error when restarting the server, comment the include of root-servers. On include in this way :

.. code::

   view "any"
   {
	match-clients { any; };
	zone "."
	{
	type master;
	file "/etc/bind/db.root";
	};
   };

