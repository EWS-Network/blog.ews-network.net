.. link: 
.. description: Manage your security groups on EC2
.. tags: Eucalyptus, AWS, Cloud
.. date: 2013/11/03 12:25:55
.. title: Manage security groups
.. slug: manage-security-groups

On the cloud, everything is acting as a service. That's also true for security. As this is a managed platform, you will also be able to manage firewall rules easily, via API.

.. class:: alert alert-info pull-right

.. contents::

Security groups - Definition
----------------------------

.. sidebar:: Terms

   Some system and network security administrators may be lost. As there is no usual terms like DMZ, Trust or Untrust, VLAN. All of this will be represented in a logical way.

A security group is an object which represents a set of rules, incoming as outgoing. By default, any imcoming traffic is denied, and all outgoing is authorized. A security group is also a way to create complete security strategy, and represent each layer.

So here, let see what we can do.

.. tip::

   There is not any specific best practices, but, I really recommend you to be carefull with cross-group security sets

Admin security groups
---------------------

I never used the default group. So, I created a specific security group to administrate the VMs and have SSH remote access.

.. code-block:: bash

   [admin@clc-0 ~] euca-create-group local-admin -d "Group to access VMs in Local"
   [admin@clc-0 ~] euca-describe-groups
   GROUP   sg-9BE43E0C     083277484068    local-admin     Group to access VMs in Local
   GROUP   sg-7D063F3D     083277484068    default default group
   [admin@clc-0 ~] euca-authorize sg-9BE43E0C -P tcp -p 22 -s a.b.c.d/subnet
   GROUP   sg-9BE43E0C
   PERMISSION      sg-9BE43E0C     ALLOWS  tcp     22      22      FROM    CIDR    a.b.c.d/subnet
   # You now have created a new security group and allow SSH from you subnet.


Applicative security groups
---------------------------

This is the common way to use Security Groups. If you want to separate groups by their applications, you can create several groups and allow only specific groups to access to each others. Like maybe Apache and Database back-ends. So, there we are.

.. code-block:: bash

   [admin@clc-0 ~] euca-create-group web-servers -d "Web Serves only"
   GROUP   sg-EFCA418E     web-servers     Web Serves only
   [admin@clc-0 ~] euca-create-group sql-backend -d "SQL Backend"
   GROUP   sg-DBAB41E8     sql-backend     SQL Backend
   [admin@clc-0 ~] euca-authorize web-servers -P tcp -p 80 -s 0.0.0.0/0
   GROUP   web-servers
   PERMISSION      web-servers     ALLOWS  tcp     80      80      FROM    CIDR    0.0.0.0/0
   [admin@clc-0 ~] euca-authorize sql-backend -P tcp -p 3306 -o sg-EFCA418E
   GROUP   sql-backend
   PERMISSION      sql-backend     ALLOWS  tcp     3306    3306    GRPID   sg-EFCA418E     FROM    CIDR    0.0.0.0/0

Alias and targets
-----------------

You may also want to use like "Aliases" to match a group of clients as being one. To do this, you can also create severs/clients security groups. Imagine, you have a master configuration machine, which all you security groups have to access. But it can be long to add each new security group you create to be authorized to access your main's. So, you create the security group of the server and you create the client one. Now, you can assign this client security group to all the VMs which have to access the configuation server, and in the security group of the server, you only have to open access from this security group. Easier to manage, and no overlap. But : do not forget to add it ;)

