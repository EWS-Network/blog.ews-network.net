.. link: 
.. description: Generic SAN Adapter
.. tags: Eucalyptus
.. date: 2013/11/03 11:40:12
.. title: Generic SAN Adapter
.. slug: generic-san-adapter

.. class:: alert alert-info pull-right

.. contents::

As we all can not buy Eucalyptus with registration, some people contribute to community with packages and addons, like Nathanxu (https://github.com/nathanxu) who is in Eucalyptus team. There, he developed a generic san adapter I've tested on my old CX300 SAN. There is a network map

.. image:: /galleries/eucalyptus/san-adapter/LAN-SAN.jpg

As you can see, the SC and NCs are connected to the SAN's LUN directly, using scsi via FiberChannel. The LUN appears on NCs and SC as /dev/sda. (Do not forget fdisk to set block alignement).

The main goal is to access the volumes created by eucalyptus and managed by the SC, from NC, directly on the LUN and not from the SC (this allow the SC fails but the NC still have access to volumes, what could not be in das or overlay configuration).

To use this SAN "plugin", you'll need to have CLC setup but not initialized yet, and all other components setup on your different servers. There, you will have to download and compile the SAN .jar file.

.. sidebar:: Alternative download

   If you prefer, and if your platform is the good one, you also can download the .jar files directly from github, and deploy these files to the right directories.

.. code-block:: bash

   yum groupinstall "Development tools" -y
   yum install axis axis2c ant java-1.7.0-openjdk-src java-1.7.0-openjdk-javadoc java-1.7.0-openjdk-devel java-1.7.0-openjdk rampartc rampartc-debuginfo libvirt-java-devel libvirt-java libvirt-devel libcurl-devel libcurl libxml2 libxml2-devel bea-stax-api libcap-devel libcap python-pip gcc make cmake python-devel libxml2-python python-lxml apache-ivy ant-nodeps git -y

Now, clone the git repository, get in and build

.. code-block:: bash

   admin@clc-0 $> git clone https://github.com/nathanxu/gen-san-adapter.git
   admin@clc-0 $> cd gen-san-adapter/
   admin@clc-0 $> ./build.sh

It will take a little time depending on your server. If you prefer, you can also download tarballs.

Once it's done, on the SC :

.. code-block:: bash

   admin@sc-0 $> ./install_sc.sh

On each Node Controller :

.. code-block:: bash

   admin@nc-X $> ./install_nc.sh

Great. Now were close to be ready to do it. What I did too, is to copy the .jar from the SC to CLC in /usr/share/eucalyptus. Now, we can initialize our cloud and register elements (see here). Once you've registered your SC, set the block manager and device target :

.. code-block:: bash

   admin@clc-0 $> euca-modify-property -p clustername.storage.blockstoragemanager=clvm
   admin@clc-0 $> euca-modify-property -p EWS-FR-0.storage.sharedevice=/dev/sda

Now we have to configure the NC lvm config and ISCSI. So, edit lvm.conf  (/etc/lvm/lvm.conf)

.. code-block:: ini

   locking_type = 2
   locking_library = "/lib/liblvm2eucalock.so"
   volume_list = [ "sda","@*" ] # replace sda by your shared device
   # add these lines at the end of your config filetags
   {
		hosttags = 1
		@NODE-IP {} # replace by your NC's IP
   }

Now, edit /etc/iscsi/initiatorname.iscsi

.. code-block:: ini

   # Change IP by your NC's one
   InitiatorName=iqn.1994-05.com.redhat:Node-IP

Once it's done, you should restart your SC to ensure settings. Now, to test if you can create a new volume.

.. code-block:: bash

   admin@clc-o $> euca-create-volume -z <zone> --size 10 # for 10Go disk

If the drive come from creating to available, it means that everything is working properly. The EBS could be created. So now, let's do some tests with it :

- Attach the drive to a VM, format and mount it
- Unmount it and remount on a different VM
- Run a bfEBS VM

If all these tests finished correctly, there you have a good "Generic San Adapter"

** Special thanks to Nathan Xu.**
