.. link: 
.. description: Tutorial to create a bfEBS EMI for Eucalyptus
.. tags: Eucalyptus, Cloud Computing, bfEBS
.. date: 2013/11/02 21:51:12
.. title: Create a bf EBS with Eucalyptus
.. slug: create-a-bf-ebs-with-eucalyptus

There are two types of instances : Instance-Store and EBS-Backend. As you will be able to see here (http://url-vers-explication-ebs-instancestrore) EBS give you the capacity to keep your datas on SAN, to stop / start a VM without loosing datas. But there is no "3 Clicks" way to have one today. So here we are going to see an easy and sure way to create your EBS EMIs.

.. class:: alert alert-info pull-right

.. contents::

Requirements
============

Of course, you will need to have a working Eucalyptus Cloud. It seems obviouis that you have enough compute capacity to run new instances and enough hard drive free space to create EBS and snapshots. Depending on your platform, some steps may be long.

Let's check some cloud-properties and status :

.. code-block:: bash

   admin@clc-0 $>euca-describe-services -E # here we check that all the Eucalyptus components are running and registred
   admin@clc-0 $>euca-describe-properties
   admin@clc-0 $>euca-create-volume -s 5 -z cluster # here we test that EBS creation is working properly


The services must be all enabled and let's understand some cloud-properties for our needs :

- cluster.storage.maxtotalvolumesizeingb : This is the size allocated for all EBS volumes
- cluster.storage.maxvolumesizeingb : This is the maximum size of 1 EBS volume (default: 15Go)

  - *You will have to change this property if you want to create more thant 15 Go EBS volumes*

- cluster.storage.shouldtransfersnapshots : Defines if your snapshots should also be transfered in S3 Walrus. (default: True)

  - *This will allow you to "distribute" all your EMIs across regions easily. But This will take more time to create the snapshot.*

- walrus.storagemaxtotalsnapshotsizeingb : This is the maximum size allocated for all your snapshots

Original VM
===========

Most of people want to do an EBS EMI from an Instance-Store one. I am not part of these people. I prefer to fully create an instance from my own. There are ways :

- You have physical access to the NC, and the capacity to use and external medium (USB CD/DVD Reader - USB Key) to boot on and setup the OS
- You do not have physical access but you have SSH access to it and your client has graphical X Server (You can also use XMing for Windows).
- The computer you are working with supports virtualization.

First case
----------

Here, we only have to create a volume, snapshot it and register an EMI. Then run a new instance. Obviously, no OS on the EBS. But thanks to the USB Device, you can add a PCI device on the fly, and declare to boot from it. Then, do your setup. When OS setup finished, you have your VM. Stop it, snapshot the root device and re-create an EMI.

.. attention:: You will need to use the init scripts provided by Eucalyptus or create your own to have SSH-Key based access and other functions.

Before creating our VM
^^^^^^^^^^^^^^^^^^^^^^

I advice you to enable graphical access (via VNC) to the VM created by Eucalyptus. To do so, uncomment vnc section in /etc/eucalyptus/libvirt.xsl :

.. code-block:: xml

    <graphics type='vnc' port='-1' autoport='yes' keymap='en-us'/>

Restart the Node controller service (If you have running VMs on it, this **will not** stop them)

.. code-block:: bash

   sudo service eucalyptus-nc restart

Second case
-----------

We now need an Instance (an instance-store one). This one will allow us to use our EBS in the future. Will eustore, download one and run it :

.. code-block:: bash

   admin@clc-0 $>eustore-describe-images
   admin@clc-0 $>eustore-install-image --help
   admin@clc-0 $>euca-run-instances $EMI -n $number -k $KEY -t $instance-size -g $group

I do advice you to create specific Security Groups depending on your Instance usage.

Use virt-manager
^^^^^^^^^^^^^^^^

I am a shell guy, mean I do everything in shell windows, but it is also true to say that GUI have a lot of benefits. Here, what we are going to do is to use virt-manager. It is a very powerful tool to manage KVM / XEN on remote and redirect all through SSH. So, here I am on a CentOS client with graphical interface. Be sure to have ssh access to your NC.

Use ssh-keys
^^^^^^^^^^^^

For any action made by virt-manager, you will be prompted for your password. To avoid this and guarantee a full secured permanent access, use ssh-keys : generate a ssh-key for your current user, then, sync it for the root user on your NC :

.. code-block:: bash

   user@client $>ssh-keygen -t rsa -b 4096
   user@client $>ssh-copy-id -i $HOME/.ssh/id_rsa.pub root@node

Once it's done, download and setup virt-manager, then, launch it. If you plan to use your own computer (the one you are working with) to create this original VM, do so, otherwise, create a new connection to your NC.

.. note:: You will need the ISO of the Operating System you plan to install. On a NC, you should put this in /var/lib/libvirt/images with the correct access rights (libvirt:root)

.. warning:: If you plan to create a Windows EMI, get the KVM drivers and kernels. If you do not provide these to windows, you will not be able to detect virtio drives

Run our new EMI
===============

Here we are ! That's the usual way to install a new operating system. You will be able to assign the values you want to your original VM.

.. warning::

   In /etc/eucalyptus/eucalyptus.conf on your NC, you are using specific VM properties : virtio. Be sure that all devices on the VM you are creating are the same according to this configuration !

.. tip::

   Here you will be able to organize your partitions on your will. There are tens way to do so. Here is the right moment to plan your instance according to its usage.

   - Classical VM ? :

     - Create a /boot, / using most of space and some for the swap
     - Create a /boot, / using most of space and NO swap : you will use a script which on startup will use part of your ephemeral as a swap device

   - Heavy storage VM ?

     - Create a /boot, and use LVM for the sub parts : in the future, if you need more space, add EBS volumes, and extend LVM ;)

Prepare our fresh VM for Eucalyptus
===================================

Our setup is finished, and after reboot we are glad to see our system running properly. But, it is a too specific one. So here we are going to "clean" this VM to be the most generic possible. We also are going to setup all our usual packages.

On linux
--------

At startup, you system created some "rules" which come from your hardware configuration. We need to delete this, because all the instances you will create from this EMI will have different properties (i.e. the eth0 MAC address). So begin with deleting these rules :

.. code-block:: bash

   rm -rfv /etc/udev/rules.d/* # rules usually are in the same directory for most common distros. Specifyt this path according to yours.

.. warning::

   For CentOS or RedHat EMIs, remember to delete the HWADDR property in /etc/sysconfig/network-scripts/ifcfg-eth0 and set BOOTPROTO to dhcp and ONBOOT to yes

We are going to use SSH to access our VMs. So to provide a fully secured system, we are going to delete the host key files which have been generated by sshd. New ones will be generated on start by sshd.

.. sidebar:: Clean tips

   When you will be at last steps, you should also remove the logs, history, and null full in persistant logs.

.. code-block:: bash

   rm -rfv /etc/ssh/ssh_host_*

On Windows
----------

There are tools available on eucalyptus documentation website which will allow your to clean your EMI in addition to a sysprep

At that moment
--------------

Here, our VM is able to get connected to the network and you can log in with SSH. There, is a list of tools you **must** setup to use advanced EC2 scripts :

.. code-block:: bash

   root@instance $>apt-get install curl bash-completion euca2ools ntp ntpdate python-boto
   root@instance $>yum install ntp curl openssh-server openssh-clients rsync euca2ools

In addition to packages setup, I **advice** you to configure the NTP client to point onto your CLC, and set all your servers to be on UTC timezone.

.. code-block:: bash

   root@instance $>chkconfig ntpd on
   root@instance $>update-rc.d ntp defaults
   root@instance $>unlink /etc/localtime && ln -s /usr/share/zoneinfo/UTC /etc/localtime

.. code-block:: bash

   # /etc/ntp.conf on instance
   server ntp.localdomain

.. warning::

   **If you do not have all the clocks synced, CLC will deny requests and none of your script calling http://169.254.169.254/ will work**

Init scripts
------------

This is no magic when you can log on your VMs with ssh private key. A script put the public key on the server for the user you wanted to be able to log on. So, now, several choices : do your own scripts, or use existing ones. You will be able to find some scripts on https://github.com/eucalyptus/Eucalyptus-Scripts . Once you get the script you want, run it manually to be sure everything is working properly.

.. note::

   On some distros you are forced to set a root password. Reset it to none, and also add sudo rights to your user if you do not want to use the root one by default (usually used: ec2-user)

Final steps
===========

Here our VM is ready. Be sure you comply with all requirements, followed my tips ;) and cleaned it. Your last command on it can also be halt, on shut it down thanks to euca-stop-instances. On the VM is stopped, we are back to our 2 cases :

First case - Second part
------------------------

Your VM is already on an EBS. So, once it is stopped, just snapshot the volume, and register your new EMI with the snapshot id :

.. code-block::

   admin@clc-0 $>euca-register -n 'name' -d 'description' -b /dev/sda=snap-ID --root-device-name /dev/sda -a ARCH # short way
   admin@clc-0 %>euca-register -n 'name' -d 'description' -b /dev/sda=snap-ID:size:true --root-device-name /dev/sda -a ARCH -b /dev/vdb=ephemeral0 # this specify EMI size and if the volume has to be deleted on termination and add an ephemeral on /dev/vdb

Second case - Second part
-------------------------

The hard drive raw file of your VM is on the Node-Controller (or on your computer) in /var/lib/libvirt/images/ and usually uses .img extension. Get this file. Now, create 2 EBS volumes on eucalyptus. One of the same size as the hard drive of your VM and another a little bigger. Then attach both to your instance-store instance. Connect to the Instance with your ssh-key, and see if the disks were correctly detected :

.. code-block:: bash

   root@instance $>dmesg
   root@instance $>fdisk -l
   # Spot the biggest, and format it, to mount it as usual (on /mnt in our case)
   root@instance $>mkfs.ext4 -m 0 /dev/vdb
   root@instance $>mount /dev/vdb /mnt

Right now, go where your VM.img file is, and rsync it to /mnt on your MV :

.. code-block:: bash

   root@nc-0$ >rsync -e 'ssh -i $HOME/cloud-user.pem' -avzt --progress --inplace /var/lib/libvirt/images/myfirstebs.img root@vm-ipL/mnt/
   # Once it's done, **dd** the image file to the EBS of the correct size
   root@instance $>dd if=/mnt/myfirstebs.img of=/dev/vdc bs=1M

Wait the dd to be finished. Then, detach the volume, create a snapshot from it, and register it like in the previous "First case" part.


The end
=======

If your VM is running properly, you can reach it and log onto, you have your EBS backed instance !!!
So now if you plan to make new EBS backed with any other tools on it (Web Server or anything else), run a VM from the EMI you've just done,
do your stuff, register, snap it and register it (Do not forget to clean it !)

Frequently asked questions
==========================

Once you have this EMI, a lot of questions may come. Here I tried to summerise the most frequently asked.

Question
--------

*If I do changes on a VM from my EBS EMI, will all the others have the changes too ?*

Answer
------

**No**. The EBS of your Instance has been copied from the snapshot, but you are not running your VM from the snapshot itsself. To keep any change on your VM for future VMs, clean, stop, snap and register it. Then, all the new instances you will run from this newly created EMI will have your changes.

