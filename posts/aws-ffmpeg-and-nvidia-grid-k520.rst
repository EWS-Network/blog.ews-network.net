.. title: AWS, FFMPEG and NVIDIA GRID K520
.. slug: aws-ffmpeg-and-nvidia-grid-k520
.. date: 2016-08-27 14:21:47 UTC
.. tags: AWS, FFMPEG, NVIDIA
.. category: AWS
.. link:
.. description: Article about use of NVIDIA GPU on AWS for FFMPEG
.. type: text

Hi everyone,

It has been a while since I haven't written a blog post, but today I wanted to share some recent experience with my public cloud of heart and their GPU instances offering. I know that, many people probably did it and did it way better than I did, especially on Ubuntu. But as I am not Ubuntu #1 fan, and much prefer CentOS I wanted to share with you my steps and results using FFMPEG and the NVIDIA codecs to leverage GPU for video encoding and manipulations.

In a near future, I will take a look at the transcoder service in AWS, but as it doesn't meet the requirements for the entire pipe of my video's lifecycle, I am yet to determine how to leverage that service.

So, historically I wanted to use the Amazon Linux image with the drivers installed (the one with the nice NVIDIA icon). But I faced much more problems with it than I thought I'd have. Therefore, I decided to go on a basic minimal install of CentOS 7 and take it from here. And here we go !

Pre-requesites
--------------

As I just mentionned before, I use for this tutorial the official CentOS 7 image available on the AWS Marketplace. This is a minimal install, so at first I recommend to install your favorite packages as well as some of the packages coming from the Base group.

.. code-block:: bash

   yum upgrade -y; yum install wget git -y; reboot

Also, to install the NVIDIA drivers, you will have to remove the "nouveau" driver / module from the machine.

.. code-block:: bash

   vi||emacs emacs /etc/default/grub

On the GRUB_CMDLINE_LINUX line, add rd.driver.blacklist=nouveau nouveau.modeset=0

.. code-block:: cfg


   GRUB_TIMEOUT=1
   GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
   GRUB_DEFAULT=saved
   GRUB_DISABLE_SUBMENU=true
   GRUB_TERMINAL="serial console"
   GRUB_SERIAL_COMMAND="serial --speed=115200"
   GRUB_CMDLINE_LINUX="console=tty0 crashkernel=auto console=ttyS0,115200 rd.driver.blacklist=nouveau nouveau.modeset=0"
   GRUB_DISABLE_RECOVERY="true"


Now, generate the new grub configuration

.. code-block:: bash

   sudo grub2-mkconfig -o /boot/grub2/grub.cfg

Now, reboot the machine.

Install CUDA
""""""""""""

CUDA is not required to do the encode and use the latest h264_nvenc encoder available in FFMPEG. However, CUDA seems required to leverage the NVIDIA api "NVResize" which redimensions videos and uses the GPU for that. Thought, as I am not an expert, there might already be an option to do so with the nvenc encoder in the latest FFMPEG version (to be continued research).
IT IS IMPORTANT THAT YOU START WITH CUDA BEFORE INSTALLING THE NVIDIA DRIVERS !!!!

So, let's get our hands on the latest CUDA SDK : https://developer.nvidia.com/cuda-downloads

There, select Linux -> x86_64 -> CentOS -> 7 -> run file (I strongly advice to go for the run file. Much more problem with the repos).

.. code-block:: bash

   wget http://developer.download.nvidia.com/compute/cuda/7.5/Prod/local_installers/cuda_7.5.18_linux.run

From your AWS EC2 Instance, this won't take long. Grab a tea :)
Now we have it, execute the run file as root

.. code-block:: bash

   sudo chmod +x cuda_7.5.18_linux
   sudo ./cuda_7.5.18_linux.run

Accept the terms, then the options I took were :

Install CUDA
Install the Libraries
Do not install the samples
The CUDA installer will install some NVIDIA drivers. At the end of the process (if successful), you should be able to enable the nvidia kernel modules :

.. code-block:: bash


   sudo modprobe nvidia sudo modprobe nvidia_uvm

So far so good, with lsmod you should see those enabled.

The CUDA utils
""""""""""""""

Thanks to this guide, I realized there would be potential steps to have some CUDA library to help FFMPEG to communicate with CUDA. Pretty straight forward step.

.. code-block:: bash

   wget http://developer.download.nvidia.com/compute/redist/ffmpeg/1511-patch/cudautils.zip unzip cudautils.zip cd cudautils make

All set for now, moving on.

The Right NVIDIA drivers
""""""""""""""""""""""""

To save you lots of troubles, let's just say that after 24h of different annoying non-verbose errors, I figured something was wrong with the drivers delivered by the CUDA installer (v.352.79). So, now, let's get the NVIDIA latest drivers. You can always get the latest from NVIDIA website : http://www.nvidia.com/download/driverResults.aspx/106780/en-us  Those are the latest (on Aug. 2016, v.367.44). Make sure you download the Linux x64 version for the GRID K520.
On your instance :

.. code-block:: bash

   wget http://us.download.nvidia.com/XFree86/Linux-x86_64/367.44/NVIDIA-Linux-x86_64-367.44.run chmod +x NVIDIA-Linux-x86_64-367.44.run sudo ./NVIDIA-Linux-x86_64-367.44.run

Accept the terms, acknowledge that this installer will install new drivers and uninstall the old ones.
I did not pick yes for the compatibility drivers for 32bits, and went for the DKMS. Once the driver install is finished, I strongly suggest to reboot, and then, as before, check on the kernel modules to verify those are enabled and working (use lsmod).

The nvEncodeAPI.h
"""""""""""""""""

You will need this header file in your library to compile FFMPEG with the --enable-nvenc and use the encoder. To get this one, you will need to subscribe to the developer program of NVIDIA and get the Video_Codec_SDK_7.0.1. I could have made this available for all, but, I will leave you to accept the terms and conditions and get your hands on it yourself.
Once you have it, upload it (via SFTP mostlikely) to your instance. Once you have it, unzip the file, and locate the nvEncodeAPI.h file. Keep it in your back-pocket, we will need it soon.

Compile FFMPEG
--------------

Now arriving on the final step. As the guide referenced earlier mentions, a few steps are required prior to compile FFMPEG :
get the right ffmpeg version
get the patch to enable the nvresize
For my own FFMPEG, I needed some additional plugins. Here is the script I used to install those. Note the exit 1 just before FFMPEG. This was a fail safe to avoid forgetting some of the little details that follow. Use the script for the non-in repos packages (ie: for x264).
Prepare your compilation folders

I personally like to put extra, self-compiled packages in /opt as they are easy to find. But feel free to do as you prefer. For the following steps, I will be doing all the work as root (I know, I know ..) in /opt (If you used my script so far, skip the folders creation).
mkdir -p /opt/ffmpeg_sources mkdir -p /opt/ffmpeg_build

Now, we can go ahead with building all the dependencies. The shell script I have done will cover those parts.

Download the right FFMPEG
"""""""""""""""""""""""""

.. code-block:: bash

   wget http://developer.download.nvidia.com/compute/redist/ffmpeg/1511-patch/ffmpeg_NVIDIA_gpu_acceleration.patch
   git clone git://source.ffmpeg.org/ffmpeg.git git reset --hard b83c849e8797fbb972ebd7f2919e0f085061f37f
   git apply --reject --whitespace=fix ../ffmpeg_NVIDIA_gpu_acceleration.patch


From this point, you can't use ./configure with --enable-nvresize and --enable-nvenc: we are missing the libraries.

nvEncodeAPI.h
"""""""""""""

Simply copy the headers file in /opt/ffmpeg_build/include
cudautils

Go back to your cuda utils folder. I did the quick and dirty, yet working, cp * /opt/ffmpeg_build/include and cp * /opt/ffmpeg_build/lib. Now, you could just put the .a in the lib and the .h in the include folders.
Configure

.. code-block:: bash


   cd /opt/ffmpeg_sources/ffmpeg/
   export PKG_CONFIG_PATH="/opt/ffmpeg_build/lib/pkgconfig"
   ./configure --prefix="/opt/ffmpeg_build" \
		--extra-cflags="-I/opt/ffmpeg_build/include" \
		--extra-ldflags="-L/opt/ffmpeg_build/lib" \
		--bindir="/opt/bin" \
		--pkg-config-flags="--static" \
		--enable-gpl \
		--enable-libfaac \
		--enable-libfreetype\
		--enable-libmp3lame \
		--enable-libtheora \
		--enable-libvorbis \
		--enable-libvpx \
		--enable-libx264 \
		--enable-nonfree \
		--enable-nvenc \
		--enable-nvresize \
		--enable-libfribidi \
		--enable-libfontconfig


If the configure succeeded, let's compile

.. code-block:: bash

   make -j make install

Now, the sneaky library missing to run it 100% requires you to deploy it to /usr/lib64 :

.. code-block:: bash

   sudo ln -s $BUILD_DIR/ffmpeg_build/lib/libfaac.so.0 /usr/lib64/
   export PATH=$PATH:/opt/bin
   ffmpeg -version ffmpeg -encoders | grep nv

If something doesn't work to run FFMPEG at this point, something went wrong before.
Test FFMPEG with NVENC and NVResize

Well for this part, I have followed the basic demo and test commands that this PDF guide suggested. So see the CPU and GPU usage, I had side-by-side in my TMUX sessions running htop and nvidia-smi (watch -d -n1 nvidia-smi).
Now in a 3 part of my TMUX, I ran the different commands, such as :

.. code-block:: bash

   ffmpeg -y -i Eucalyptus-Ansible-Deploy.mp4 -filter_complex nvresize=5:s=hd1080\|hd720\|hd480\|wvga\|cif:readback=0[out0][out1][out2][out3][out4] -map [out0] -acodec copy -vcodec nvenc -b:v 5M out0nv.mkv -map [out1] -acodec copy -vcodec nvenc -b:v 4M out1nv.mkv -map [out2] -acodec copy -vcodec nvenc -b:v 3M out2nv.mkv -map [out3] -acodec copy -vcodec nvenc -b:v 2M out3nv.mkv -map [out4] -acodec copy -vcodec nvenc -b:v 1M out4nv.mkv ffmpeg -y -i Eucalyptus-Ansible-Deploy.mp4 -acodec copy -vcodec nvenc -b:v 2M /var/tmp/test_parallel.mp4


Tada ! I hope this guide will have helped you guys in your different experimentation with AWS and will enjoy playing around with it.

.. note::

   Do not forget to stop/terminate those instances. A potential 600USD bill will wait for your per GPU instance running. Still less than traditional IT but still .. :P
