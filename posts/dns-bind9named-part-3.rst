.. link:
.. description: DNS - Bind9/Named
.. tags: Linux, DNS, Bind9, Named
.. date: 2013/11/03 14:01:35
.. title: DNS - Bind9/Named [Part 3]
.. slug: dns-bind9named-part-3

Right now we have a really nice configuration of bind9. But we have no log about queries and actions. So what we're going to do now is to log in a file different queries and actions. To do we gonna add some lines in named.conf.local (or in options, if you prefer).

.. code::

   logging {
   channel query.log
   {
	file "/var/log/bind/query.log";
	//Set the severity to dynamic to see all the debug messages.
	severity debug 3;
   };
	category default { default_syslog; default_debug; };
	category unmatched { null; };
	category queries { query.log; };
   };
