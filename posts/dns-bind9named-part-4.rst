.. link:
.. description: DNS - Bind9/Named
.. tags: Linux, DNS, Bind9, Named
.. date: 2013/11/03 14:01:41
.. title: DNS - Bind9/Named [Part 4]
.. slug: dns-bind9named-part-4

Ok now we have a DNS server with all queries logged, and some ACLs management. But what if this server fails ? To provide high availability, we're going to setup a mastery-slave DNS system. Of course, do not forget to add the second DNS in your clients configuration.

So on the master server, we are going to create a key we will use to authenticate updates :

.. code-block:: bash

   dnssec-keygen -a hmac-md5 -b 256 -n host somewhere.net

Once the key is generated, get the key.

.. code-block:: bash

   cat <yourkey>.private | grep Key

We are going to create a new file in which we will add our server set : ns-lan.conf

.. code::

   key "LAN-TRANSFER"
   {
	algorithm hmac-md5;
	secret "<your key here>";
   };
   server 192.168.1.1
   {
	keys { LAN-TRANSFER; };
   };
   server 172.16.1.1
   {
	keys { LAN-TRANSFER; };
   };

Ok so now bind is able to identify each server and the key assigned to it. Copy the key and configuration file on your slave server. Now, I will assume that your secondary server has the same original named.conf.local and db.somewhere.net. Let's see the local config file, and edit some parts

.. code::

   zone "somewhere.net"
   {
	type slave;
	file "/etc/bind/db.somewhere.net";
	masters { 192.168.1.1; };
	allow-notify { 192.168.1.1; };
   };

Here our server is now serving the DNS zone as a slave. And if your make an update on the master, the slave server will be notified about it and be updated. (Do not forget to give bind write access on /etc/bind). Now to test it, that's very easy : edit your master server zone config file, and thanks to the logs, see if the slave received the notification. After that, stop the master server and try to ping (or use dig - more advanced) to see if the slave server is able to give you the correct answer.

For a "master-master" configuration, add on your masters the instruction :

.. code::

   also-notify { sec-master-ip; };"
