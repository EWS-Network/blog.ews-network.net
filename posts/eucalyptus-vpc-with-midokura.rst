.. title: Eucalyptus VPC with Midokura
.. slug: eucalyptus-vpc-with-midokura
.. date: 2015-03-09 00:42:00 UTC
.. tags: Eucalyptus, VPC, Private Cloud, Midokura, SDN
.. category: Eucalyptus
.. link:
.. description:
.. type: text

VPC
===

In 2014 VPC became the default networking mode in AWS letting the EC2 Classic networking mode go. VPC is a great way to manage and have control over the network environment into which the AWS resources will run. It also gives full control in the case of an hybrid cloud or at least in the case of your IT extension, with a lot of ways to interconnect the two.

A lot of new features came out from this but most importantly, VPC would provide the ability for everyone to have backend applications running in Private. No public traffic, no access to and from the internet unless wanted. A keystone for AWS to promote the Public cloud as a safe place.

Midokura
========

Midokura is a SDN software which is used to manage routing between instances, to the internet, security groups etc. The super cool thing about about Midokura is its capacity to be high-available and scalable in time. Of course being originally a networking guy, I also find super cool to have BGP capability.

Requirements
============

Here is what my architecture looks like :

.. thumbnail:: /images/euca-and-vpc/vpc-0002.jpg

VLAN 1001 is here for Eucalyptus components communication. VLAN 1010 is here for Midonet components communication including Cassandra and Zookeeper. VLAN 1002 is our BGP zone. VLAN 0 is basically the ifaces default network which is not relevant here. We will use it only for packages download etc.

From here, we need :

- UFS (CentOS 6)
- CLC (CentOS 6)
- CCSC (CentOS 6)
- NC(s) (CentOS 6)
- Cassandra/Zookeeper  (CentOS 7)
- BGP Server / Router

Of course, you could use all on the same L2/L3 network. But you are not a bad person, are you ? ;) For the Eucalyptus components we will be on CentOS 6.6, CentOS 7 for the others. Some of the Eucalyptus components will have a midonet component depending on the service they will be running. Today we will do a single-cluster deployment, but nothing will change in that regard.

But before going any further, we should sit and understand how Midokura and Eucalyptus will work together. So, Midokura is here as a SDN provider. Eucalyptus is here to understand VPC / EC2 etc API calls and pilot the other components to provide the resources.
Now, what is VPC ? VPC stands for Virtual Private Cloud. Technically what it means ? You will be able to create a delimited zone (define the perimeter) specifying the different networks (subnets) in which instances will be running. By default, those instance will have a private IP address only, and no internet access unless you specifically give it.

In a classical envirnoment, that would correspond to have different routers (L3) connected to different Switches responsible for traffic isolation (L2). Here, this is exactly what Midokura will do for us. Midokura will create virtual routers and switches which will be used by Eucalyptus to place resources.

How Eucalyptus and Midokura work together - Logical view

I am on my Eucalyptus cloud. No VPC has been created. At this time, I have no resources created. Eucalyptus will have created on midokura 1 router, called "eucart". This router is the "top-upstream" router, to which new routers will be created. For our explanation, we will call this router EUCART.

So now, when I create a new VPC, I do it with

.. code-block:: bash

   euca-create-vpc CIDR
   euca-create-vpc 192.168.0.0/24

This in the system will create a new router, which we will call RouterA. This routerA will be responsible for the communication of instances between subnets. But at this time, I have no subnets. So, let's create two :

.. code-block:: bash

   euca-create-subnet -c VPC_ID -i CIDR -z AZ
   euca-create-subnet -c VPCA -i 192.168.0.0/28 -z euca-az-01
   euca-create-subnet -c VPCA -i 192.168.0.16/28 -z euca-az-01

Now I have two different subnets. If I had to represent the network we have just created we would have :

.. thumbnail:: /images/euca-and-vpc/vpc-0003.jpg

Of course, if I had multiple VPC, the we would simply have duplicated VPCA group, and more switchs if we had more subnets.

As in AWS, you can have an "internet gateway" which is simply a router which will all instances to have public internet access, as soon as they have an Elastic IP (EIP) associated with those.

Here it is for a few logical mechanism. Let's attack the technical install. Brace yourself, it is going go take a few times.

Eucalyptus & Midokura - Install
===============================

First we are going to begin with Eucalyptus. I won't dig too much on the steps you can find `here <http://eucalyptus.com/docs>`_ for the packages install.
As said, we gonna have in this deployment : 1 CLC, 1UFS, 1 CC/SC and 1 NC. But before initializing the cloud, we are going to do a few VLANs to separate our traffic.

Servers Network configuration
-----------------------------

.. code-block:: bash

   vconfig add <iface> <vlan number>
   vconfig add em2 1000
   ifconfig em2.1000 192.168.1.XXX/26 # change the IP for each component

For the NC
~~~~~~~~~~

.. note::  Don't assign the IP on the VLAN iface

.. code-block:: bash

   vconfig add em2 1000
   ifconfig em2.1000 up
   brctl addbr br0
   brctl addif br0 em2.1000


Eucalyptus network Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From here, make sure all components can talk to each other.
Then, for our CLC, UFS and SC we are going to specify that they have to use the VLAN iface in eucalyptus.conf

.. code-block:: bash

   vi /etc/eucalyptus/eucalyptus.conf
   CLOUD_OPTS="--bind-addr=192.168.1.XX" # here, use your machine's IP as set previously


.. code-block:: bash

   euca_conf --initialize

While it is initializing, we are going to add on our CLC and NC a new VLAN, as well as on the Cassandra and Zookeeper machines. For those, I will use VLAN 1001

.. code-block:: bash

   vconfig add em2 1001
   # here I am putting my machines in a different subnet than for VLAN 1000 | change for each machine
   ifconfig em2.1001 192.168.1.65/28 up

Cassandra - Zookeper
--------------------

Alright, let's move onto the Cassandra / Zookeeper machine. In this deployment I will have only 1 machine to host the cluster, but of course for production, the minimum recommended is 3 to have a better scale and redundancy capacity.


Repositories
~~~~~~~~~~~~

*File datastax.repo*

.. code-block:: ini

   # DataStax (Apache Cassandra)
   [datastax]
   name = DataStax Repo for Apache Cassandra
   baseurl = http://rpm.datastax.com/community
   enabled = 1
   gpgcheck = 0
   gpgkey = https://rpm.datastax.com/rpm/repo_key

*File midonet.repo*

.. code-block:: ini

   [midonet]
   name=MidoNet
   baseurl=http://repo.midonet.org/midonet/v2015.01/RHEL/7/stable/
   enabled=1
   gpgcheck=1
   gpgkey=http://repo.midonet.org/RPM-GPG-KEY-midokura

   [midonet-openstack-integration]
   name=MidoNet OpenStack Integration
   baseurl=http://repo.midonet.org/openstack-juno/RHEL/7/stable/
   enabled=1
   gpgcheck=1
   gpgkey=http://repo.midonet.org/RPM-GPG-KEY-midokura

   [midonet-misc]
   name=MidoNet 3rd Party Tools and Libraries
   baseurl=http://repo.midonet.org/misc/RHEL/7/misc/
   enabled=1
   gpgcheck=1
   gpgkey=http://repo.midonet.org/RPM-GPG-KEY-midokura


Packages install
~~~~~~~~~~~~~~~~

.. code-block:: bash

   yum install zookeeper zkdump cassandra21 java-1.7.0-openjdk-headless

Once it is installed, we need to configure the runtime environment for those applications. First, Zookeeper.


Zookeeper configuration
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   mkdir -p /usr/java/default/bin/
   ln -s /usr/lib/jvm/jre-1.7.0-openjdk/bin/java /usr/java/default/bin/java
   mkdir /var/lib/zookeeper/data
   chmod 777 /var/lib/zookeeper/data

We also need to add a configuration line in zoo.cfg
*File /etc/zookeeper/zoo.cfg*

.. code-block:: ini

   server.1=<VLAN 1001 IP>:2888:3888

We can now indicate which server ID we are and start the service:

.. code-block:: bash

   echo 1 > /var/lib/zookeeper/data/myid
   systemctl start zookeeper.service


Cassandra configuration
~~~~~~~~~~~~~~~~~~~~~~~

Cassandra is a bit easier to install and configure. Simply replace a few values into the configuration files

*File /etc/cassandra/cassandra.yaml*

.. code-block:: yaml

   cluster_name: 'midonet'
   rpc_address: <VLAN 1001 IP>
   seeds: <VLAN 1001 IP>
   listen_address: <VLAN 1001 IP>

Now we clean it up and start the service:

.. code-block:: bash

   rm -rf /var/lib/cassandra/data/system/
   systemctl restart cassandra.service

And this is :) Of course, don't forget to open the ports on your FW and / or local machine.

Eucalyptus configuration
------------------------

In the meantime, our cloud has been initialized. Now simply register the services as indicated `in the documentation - register components <https://www.eucalyptus.com/docs/eucalyptus/4.1.0/index.html#install-guide/registering_components.html>`_. Don't forget to use the VLAN 1000 IP address for registration.

For all components, don't forget to change the VNET_MODE value to "VPCMIDO" instead of "MANAGED-NOVLAN" which will indicate to the components that their configuration must fit VPCMIDO requirements.

So, from here the Cassandra and Zookeeper will allow us to have midonet-API and midolman installed.
Midonet-API is here to be the endpoint against which the Eucalyptus components will do API calls to create new routers and switches, as well as configure security groups (L4 filtering). Midolman is here to connect the different systems together and make the networking possible. You MUST have a midolman on each NC and CLC. Midonet-API is only to be installed on the CLC.

To have the API working, for now in Eucalyptus 4.1.0 we have to (sadly) have it installed on the CLC and the CLC only (that is the sad thing). Here our CLC will act as the "Midonet Gateway", this EUCART router I was talking about previously.

Let's do the install : (of course, here you will also need the midonet.repo we used before).

.. code-block:: bash

   yum install tomcat midolman midonet-api python-midonetclient


Tomcat will act as server for the API (basically). Unfortunately, the port in Eucalyptus to talk to the API has been hardcoded :'( to 8080. So before going any further we need to change one of Eucalyptus' port to a different on Eucalyptus itself:

.. code-block:: bash

   $> euca-modify-property -p www.http_port=8081
   8081 was 8080

If you don't make this change, your API will never be available.

Now the packages are installed and the port 8080/TCP free, we must configure Tomcat itself to serve the midonet API. Add a new file into /etc/tomcat/Catalina/localhost named "midonet-api.xml"

*File /etc/tomcat/Catalina/localhost/midonet-api.xml*

.. code-block:: xml

   <Context
     path="/midonet-api"
     docBase="/usr/share/midonet-api"
     antiResourceLocking="false"
     privileged="true"
   />

Good, so now we need to configure Midonet API to get connected to our Zookeeper server. Go into /usr/share/midonet-api/WEB-INF and edit the file web.xml

.. code-block:: xml

   <context-param>
		<param-name>rest_api-base_uri</param-name>
		<param-value>http://<CLC VLAN 1001 IP>:8080/midonet-api</param-value>
   </context-param>

   # [...]
   <param-name>auth-auth_provider</param-name>
   <!-- Specify the class path of the auth service -->
		<param-value>
		# old value is for Keystone OS
		# new value ->
		org.midonet.api.auth.MockAuthService
		</param-value>
   </context-param>

   <context-param>
		<param-name>zookeeper-zookeeper_hosts</param-name>
		<!-- comma separated list of Zookeeper nodes(host:port) -->
		<param-value><ZOOKEEPER VLAN 1001 IP>:2181</param-value>
  </context-param>


Alright, now we can start tomcat which will enable the midonet-api. To verify, you can simply do a curl call on the entry point

.. code-block:: bash

   curl <CLC VLAN 1001 IP>:8080/midonet-api/

Midolman
--------

We can configure midolman. The good thing about the midolman configuration, is that you can use the same configuration across all nodes. Once more, we simply have to change a few parameters to use our Cassandra / Zookeeper server. Edit /etc/midolman/

.. code-block:: ini

   [zookeeper]
   #zookeeper_hosts = 192.168.100.8:2181,192.168.100.9:2181,192.168.100.10:2181
   zookeeper_hosts = <ZOOKEEPER VLAN 1001 IP>:2181
   session_timeout = 30000
   midolman_root_key = /midonet/v1
   session_gracetime = 30000

   [cassandra]
   #servers = 192.168.100.8,192.168.100.9,192.168.100.10
   servers = <CASSANDRA VLAN 1001 IP>:9042
   # DO CHANGE THIS, recommended value is 3
   replication_factor = 1
   cluster = midonet


This is it, nothing else to configure.

.. warning:: you need to have IP route with netns installed. To verify, simply try "ip netns list". If you end with an error, you need to install the `iproute netns package <https://repos.fedorapeople.org/repos/openstack/openstack-havana/epel-6/iproute-2.6.32-130.el6ost.netns.2.x86_64.rpm">`_.

Now we are done with the config files, we can start the services. For Midolman, there is not default init.d script installed. So, here it is :

*File /etc/init.d/midolman*

.. code-block:: bash

   #!/bin/bash
   #
   # midolman      Start up the midolman virtual network controller daemon
   #
   # chkconfig: 2345 80 20
   #
   ### BEGIN INIT INFO
   # Provides: midolman
   # Required-Start: $network
   # Required-Stop: $network
   # Description:  Midolman is the virtual network controller for MidoNet.
   # Short-Description: start and stop midolman
   ### END INIT INFO
   #
   #
   # Midolman's backwards compatibility script to forward requests to upstart.
   # Based on Ubuntu's /lib/init/upstart-job


   set -e
   JOB="midolman"
   if [ -z "$1" ]; then
   echo "Usage: $0 COMMAND" 1>&2
				   exit 1
   fi
   COMMAND="$1"
   shift
   case $COMMAND in

    status)
        status_output=`status "$JOB"`
	echo $status_output
	echo $status_output | grep -q running
	;;
    start|stop)
	if status "$JOB" 2>/dev/null | grep -q ' start/'; then
	    RUNNING=1
	fi
	if [ -z "$RUNNING" ] && [ "$COMMAND" = "stop" ]; then
	    exit 0
	elif [ -n "$RUNNING" ] && [ "$COMMAND" = "start" ]; then
	    exit 0
	elif [ -n "$DISABLED" ] && [ "$COMMAND" = "start" ]; then
	    exit 0
	fi
	$COMMAND "$JOB"
	;;
    restart)
	if status "$JOB" 2>/dev/null | grep -q ' start/'; then
	    RUNNING=1
	fi
	if [ -n "$RUNNING" ] ; then
	    stop "$JOB"
	fi

	# If the job is disabled and is not currently running, the job is
	# not restarted. However, if the job is disabled but has been forced into the
	# running state, we *do* stop and restart it since this is expected behaviour
	# for the admin who forced the start.

	if [ -n "$DISABLED" ] && [ -z "$RUNNING" ]; then
	    exit 0
	fi
	start "$JOB"
	;;
    reload|force-reload)
	reload "$JOB"
	;;
    *)
	$ECHO
	$ECHO "$COMMAND is not a supported operation for Upstart jobs." 1>&2
	exit 1
    esac


Once you have installed and configured midolman for every components, we need to configure midonet to have all our cloud component, here we will simply call them "hosts" (the terminology is very important).

Back on our CLC, let's add a midonetrc file so we don't have to specify the IP address everytime

.. code-block:: ini

   [cli]
   api_url = http://<CLC VLAN 1001 IP>:8081/midonet-api
   username = admin
   password = admin
   project_id = admin

Here, the credentials are not important and won't work. So anytime, to get onto midonet-cli, use the option "-A"

Before we get any further, there are 2 new packages which MUST be installed on the CLC : eucanetd and nginx. Explanations later.

.. code-block:: bash

   yum install eucanetd nginx -y

We are half the way. I know, sounds like quite a lot. But in fact, that is not that much. We now need to configure Eucalyptus network configuration. This, as for EDGE, is done using a JSON template. Pay attention, a mistake will cause you headaches for a long time.

.. code-block:: javascript

   {
     "InstanceDnsServers":
     [
      "10.1.1.254"
     ],
     "Mode": "VPCMIDO",
     "PublicIps":
     [
       "172.16.142.10-172.16.142.250"
     ],
     "Mido": {
       "EucanetdHost": "odc-c-30.prc.eucalyptus-systems.com",
       "GatewayHost": "odc-c-30.prc.eucalyptus-systems.com",
       "GatewayIP": "172.16.128.100",
       "GatewayInterface": "em1.1002",
       "PublicNetworkCidr": "172.16.0.0/16",
       "PublicGatewayIP": "172.16.255.254"
     }
   }

So here, what does it mean ?

- InstanceDnsServers : The list of nameservers. Nothing unexpected
- Mode : VPCMIDO : Indicates to the cloud that VPC is enabled
- PublicIps : List of ranges and / or subnets of Public IPs which can be used by the instances
- Mido : This is the most important object !!
- EucanetdHost: String() which points onto the server which runs the eucanetd binary and midonetAPI
- GatewayHost: String() which points onto the server which runs the midonet GW. As said for now the GW and EucanetdHost must be the same machine.
- GatewayIP : String() which indicates which IP will be used by the router EUCART. Here, you must use an IP address which DOESNT EXIST !!!
- GatewayInterface : The IFACE which is used for the GatewayIP. Here, I had created a dedicated VLAN for it, vlan 1002.
- PublicNetworkCidr: String() which is the network / subnet for all your public IPs. Here in my example, I am using a /16 and defined only a /24 for my cloud public IPs. It is because I can have multiple Clouds in this /16 which each will use a different range of IPs
- PublicGatewayIP : String() which points on our GBP router.

.. warning:: Don't forget that the GatewayInterface must be an interface WITHOUT an IP address set


For now in 4.1.0 as VPC is techpreview, many configuration and topologies are not yet supported. So for now, you must keep the MidonetGW on the CLC and have the EucanetdHost and EucanetdHost pointing onto the CLC DNS' name. And this MUST be a DNS name, otherwise the net namespaces wont be created correctly.

Also as we speak of DNS, if those VLAN we created can lead to resolve the hostname, you MUST add in your local hosts file the VLAN1001 IP to resolve your hostname.

Alright, at this point we can have instances created into subnets, but they won't be able to get connected to external networks. We need to setup BGP and configure midonet for that.

Get onto the the BGP server. Here, we are only going to create 1 VLAN, which we will use for public Addresses of instances. Here, we gonna use 172.16.0.0/16 and our BGP router will use 172.16.255.254 as we indicated into the JSON previously.

.. code-block:: bash

   vconfig add em1 1002
   ifconfig em2.1002 172.16.255.254 up

To have it working, it is very easy : (originally I followed `this tutorial <http://xmodulo.com/centos-bgp-router-quagga.html">`_)

.. code-block:: bash

   yum install quagga
   setsebool -P zebra_write_config 1

Now the vty config itself and BGP are really simple :

*File /etc/quagga/bgpd.conf*

.. code-block::

   ! -*- bgp -*-
   !
   ! BGPd sample configuratin file
   !
   ! $Id: bgpd.conf.sample,v 1.1 2002/12/13 20:15:29 paul Exp $
   !
   hostname bgpd
   password zebra
   !enable password please-set-at-here
   !

   router bgp 66000
   bgp router-id 172.16.255.254
   network 172.16.0.0/16
   neighbor 172.16.128.100 remote-as 66742
   neighbor 172.16.128.101 remote-as 66743

   !
   log file bgpd.log
   !

Here we can see that the server will get BGP information from 2 "neighbor" with unique IDs. We will later be able to have 1 peer per midonet GW which will be used by the system to reach networks.

To simplify : the BGP server is waiting for information coming from other BGP servers. Those BGP servers will be our MidonetGW. Our MidonetGW will then announce themselves to the server saying "hi, I am server ID XXXX, and I know the route to YY networks". Once the announcement is done on the root BGP router, all traffic going to it to reach our instances EIP will be sent onto our MidonetGW.

Here is the zebra config.

.. code-block::

   cat /etc/quagga/zebra.conf
   !
   ! Zebra configuration saved from vty
   !   2015/03/05 13:14:09
   !
   hostname Router
   password zebra
   enable password zebra
   log file /var/log/quagga/quagga.log
   !
   interface eth0
   ipv6 nd suppress-ra
   !
   interface eth0.1002
   ipv6 nd suppress-ra
   !
   interface eth1
   ipv6 nd suppress-ra
   !
   interface lo
   !
   !
   !
   line vty
   !

Alright, almost finished !.
Back on our CLC, we need to configure the EUCART router.

.. code-block:: bash

   # we list routers
   midonet> router list
   router router0 name eucart state up

   # we list the ports on the router
   midonet> router router0 list port
   port port0 device router0 state up mac ac:ca:ba:b0:df:d8 address 169.254.0.1 net 169.254.0.0/17 peer bridge0:port0
   port port1 device router0 state up mac ac:ca:ba:09:b9:47 address 172.16.128.100 net 172.16.0.0/16

   # At this point, we know that we must configure port1 as it has the GWIpAddress we have set in the JSON earlier. Check if there is any BGP configuration done on it

   midonet> router router0 port port1 list bgp
   # nothing, it is normal we have not set anything yet
   # first we need to add the BGP peering configuration

   midonet> router router0 port port1 add bgp local-AS 66742 peer-AS 66000 peer 172.16.255.254
   # here, note that the values are the same as in bgpd.conf . Our router is ID 66742 where the root BGP is 66000

   # Now, we need to indicate that we are the routing device to our public IPs

   router router0 port port1 bgp bgp0 add route net 172.16.142.0/24
   # in my JSON config I have used only 240 addresses, but those addresses fit into this subnet
   # ok, at this point, things can work just fine. Last step is to indicate on which port the BGP has to be
   # to do so, we need to spot the CLC VLAN 1002 interface

   midonet> host list
   host host0 name odc-c-33.prc.eucalyptus-systems.com alive true
   host host1 name odc-c-30.prc.eucalyptus-systems.com alive true

   # Now we create a tunnel zone for our hosts
   tunnel-zone create name euca-mido type gre
   # Add the hosts
   tunnel-zone list
   tunnel-zone tzone0 add member host host0 address A.B.C.D
   tunnel-zone tzone0 add member host host1 address X.Y.Z.0

   # here my GW is host1

   midonet> host host1 list interface
   iface midonet host_id host1 status 0 addresses [] mac 9a:3a:bd:6d:89:c2 mtu 1500 type Virtual endpoint DATAPATH
   iface lo host_id host1 status 3 addresses [u'127.0.0.1', u'0:0:0:0:0:0:0:1'] mac 00:00:00:00:00:00 mtu 65536 type Virtual endpoint LOCALHOST
   iface em2.1001 host_id host1 status 3 addresses [u'192.168.1.67', u'fe80:0:0:0:baac:6fff:fe8c:e96d'] mac b8:ac:6f:8c:e9:6d mtu 1500 type Virtual endpoint UNKNOWN
   iface em1.1002 host_id host1 status 3 addresses [u'fe80:0:0:0:baac:6fff:fe8c:e96c'] mac b8:ac:6f:8c:e9:6c mtu 1500 type Virtual endpoint UNKNOWN
   iface em2.1000 host_id host1 status 3 addresses [u'192.168.1.3', u'fe80:0:0:0:baac:6fff:fe8c:e96d'] mac b8:ac:6f:8c:e9:6d mtu 1500 type Virtual endpoint UNKNOWN
   iface em1 host_id host1 status 3 addresses [u'10.104.10.30', u'fe80:0:0:0:baac:6fff:fe8c:e96c'] mac b8:ac:6f:8c:e9:6c mtu 1500 type Physical endpoint PHYSICAL
   iface em2 host_id host1 status 3 addresses [u'10.105.10.30', u'fe80:0:0:0:baac:6fff:fe8c:e96d'] mac b8:ac:6f:8c:e9:6d mtu 1500 type Physical endpoint PHYSICAL

   # iface em1.1002 -> no IP address, good. We can now bind the router onto it.
   midonet> host host1 add binding port router0:port1 interface em1.1002


On your CLC, you should see new interfaces being created called "mbgp_X". This is good sign. This means that your BGP processes are running and broadcasting information. Let's check on the upstream BGP that we have learned those routes.

.. code-block:: bash

   vtysh

   Hello, this is Quagga (version 0.99.22.4).
   Copyright 1996-2005 Kunihiro Ishiguro, et al.

   router# show ip bgp
   BGP table version is 0, local router ID is 172.16.255.254
   Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
   r RIB-failure, S Stale, R Removed
   Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
   * 172.16.0.0       0.0.0.0                  0         32768 i
   * 172.16.142.0/24  172.16.128.100           0         0 66742 i

   Total number of prefixes 2

Here we can see that we have router 66742 which has announced he knows the route to the subnet 172.16.142.0/24.

Now on our cloud, if we get an EIP on the instances, we will be able to reach those instances and / or host services accessible from - potentially - anywhere.
