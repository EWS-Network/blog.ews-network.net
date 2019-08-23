.. link: 
.. description: DNS with Bind9 / Named
.. tags: Linux, Bind9, Named, DNS
.. date: 2013/11/03 13:21:17
.. title: DNS - Bind9/Named [Part 1]
.. slug: dns-bind9named-part-1

The most popular DNS server is bind9 also called "named" (bind on Debian and Named on RedHat). There are a lot of tutorials on the internet to learn you how to do crazy things, you will find it easily. But you can spend hours to find what you want. So, here I'm going to see with you the setup and configuration of bind.

In this tutorial, I will be working on Debian 7, but all configuration files of bind will also work on RedHat (the paths will change).

What are we going to see ?

- Setup DNS server
- Basic zone configurations
- Basic server configurations

Install DNS Server
------------------

.. code-block:: bash

   # Debian
   user@nameserver $> apt-get update ; apt-get install bind9 dnsutils -y
   # RedHat
   user@nameserver $> yum update ; yum install bind bind-utils -y

Configuration
-------------

Ok now bind is setup, we're going to configure it. In our examples, we will use the domain name "somewhere.net". In Debian, the config files are in /etc/bind and in RedHat the files are in /etc and /etc/named/. As ever, I do advice you to backup it somewhere to roll-back.

We will also find db. files, which are default or examples configuration files. We're going to copy the db.empty do db.<domain_name>
**db.somewhere.net.** In this file, we modify localhost with our values :

.. sidebar:: Main files

   :named.conf:
      Main config file (the only one in RedHat, I advice you to split it in different files)

   :named.conf.local:
      The configuration file where we gonna set the properties of our zones

   :named.conf.options:
      The options config file

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

Here we have also set some little parameters in our db.file. We're now going to use it in bind to be able to resolve it.Open to edit named.conf.local. Here, we're going to add our zone.

.. code::

   zone "somewhere.net"
   {
	type master;
	file "/etc/bind/db.somewhere.net";
   };

Ok so there we have a very basic domain configuration. But our server is not going to serve this domain until you allowed it. So now let's see the bid configuration. We have a file called named.conf.options - In this file we can find some examples and default config. So here, what we gonna do is to allow clients to query the server, and thing which is really useful, use our DNS as a forwarder (for non-localy hosted domains) :

.. code::

   [...]
   forwarders { 8.8.8.8; };
   allow-query { any; };
   allow-query-cache { localnet; };
   allow-recursion { localhost; };
   [...]

Here we said to our server to allow queries from anywhere. Maybe you want your server to answer only to your local machines. So to do this, we gonna use the ACLs, which we will see in next part. Common keywords are :

- any
- localnet
- localhost

So now if we restart our server, you will be able to query it from clients.
