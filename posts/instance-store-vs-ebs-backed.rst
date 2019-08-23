.. link:
.. description: EBS vs Instance-Store
.. tags: Eucalyptus, AWS, Cloud
.. date: 2013/11/04 17:06:17
.. title: Instance-Store vs EBS-backed
.. slug: instance-store-vs-ebs-backed

.. class:: alert alert-info pull-right

.. contents::

What is that ?
==============

On Eucalyptus, and also on Amazon Web Services (AWS) there are two types of backed instances : instance-store and EBS. Each one has its advantages and drawbacks, and what we are going to see here is which they are, and how can we use it the most efficiently.

If you've just setup a new Eucalyptus cloud using the fast-start (http://www.eucalyptus.com/eucalyptus-cloud/get-started/try/faststart) you will follow the instructions and finally arrive at this point : run a VM. So here we are

Instance-Store
--------------

.. sidebar:: Instance-Store creation

   .. figure:: /galleries/eucalyptus/ebs-istores/instance-stores.png
      :align: left
      :alt: Instance store creation diagram

      Here we can see the EMI Bucket which comes from Walrus

Instance-Store are usually the first type of VM you will add on your cloud. Very easy to use thanks to eustore (http://www.eucalyptus.com/docs/eucalyptus/3.1/ug/eustore-browse-install.html), instance-store EMIs are stored on Walrus (S3 on AWS). These EMIs are very light, with the minimal setup and packages. This is why you will be able to find images of 500Mo for a complete CentOS system. In an instance-store, there are 3 components :

- Kernel Image
- Ramdisk Image
- System Image

Obviously, the kernel and the ramdisk are here to fit your hypervisor (XEN / KVM) and still work on the same way. Then we have the image. This is your OS image. Compressed, this will be expanded to a static size once it runs in your cloud. So as this EMI is packaged in S3, this is very easy for you to have it on all your region.

Advantages
^^^^^^^^^^

- Really fast to run:

  - It is a package downloaded from the Walrus server it is really fast and your VM will start within the minute (depending on your network speed).

- Light:

  - It is a small package. This do not use a lot of disk space on your walrus.

- On demand ready :

  - With the eustore, you have access to a lot of official EMIs and the community ones.

- Ephemeral root device usually have a best IO than EBS and disk space is not paid


Drawbacks
^^^^^^^^^

- Volatile:

  - On a NC crash, all the Instances will be lost (and also the data stored on its primary hard drive)

- Harder to customize and create new EMIs from a running one, as its hard drive is deleted on shutdown, and no on the fly creation

- Cannot be stopped : on stop, instance will always go terminated

bfEBS Instances
---------------

.. sidebar:: bfEBS Creation

   .. image:: /galleries/eucalyptus/ebs-istores/bf-EBS.png

   Here we have two cases :
      - the snaphot as been stored as a snapset in a S3 Bucket
      - the snapshot is stored as a raw file on SC.

As you can see on the picture, bfEBS are instances which can be ran on any node like an instance-store one, but which boots here, not from the hard drive of the Node Controller, but from the Storage Controller EBS Volume. This will make our VM dependant on the IOps limitations. This also is a useful feature, this way you will be able to create instances with disks using different IOps according to the VM usage: low IOps for front-end VMs and high-IOps for databases or file servers.

Advantages
^^^^^^^^^^

- Fault tolreant

  - If the NC crashs, the Instance can be restarted on another NC.

- Different IOps levels
- Root device can be changed dynamically
- Snapshot the volumes
- On the fly AMI creation

Drawbacks
^^^^^^^^^

- On Eucalyptus, this may take more time in overlay/das/clvm modes (no DELL/EMC2/NetAPP SAN)
- This uses disk space which is paid on AWS

