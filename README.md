configure-hadoop-raspi
==============

Configuring a Hadoop Cluster with Raspberry Pi's

Welcome!
--------

To be clear, the following setup is impractical and should be done purely as a learning excercise. The JVM (Java Virtual Machine) which powers Hadoop is horridly slow on the ARM CPUs that drive the Raspberry Pi. Feel free to use a Model A or B Raspberry Pi as neither will be powerful enough to do any real  data crunching. This project exists solely to help in booting up your first Hadoop Environment! 

Technically you could draw more power out of the setup with a few of the following considerations, but that's beyond the scope of this guide: Hard Float OS, Oracle JDK, JVM tuning params.

Before we start
---------------

The following items are not all necessary. Obviously, you could have any combination of equipment. Technically you could run all of the hadoop tools on a single node if you wanted. By choosing these items we're just laying out the most common path we assume people will take. If anything below is specific to these items, we'll point it out.

Requirements:

- 3x Raspberry Pi Model B
- 3x 32GB SD Card
- 3x MicroUSB Power Cable + Charger
- 4x Ethernet Cable
- 1x 5 port unmanaged switch
- 1x power strip

Rack Setup:

	 power
	   |
	 -----   1 power strip
	 | | |
	 [][][]  3 nodes
	 | | |
	 -----   1 network switch
	   |
	 router

Kick each node
--------------

Raspberry Pi OS installation methods change regularly. At the time of writing this guide, NOOBS was the ease-of-use installer and so that's what we'll describe here.

- Download NOOBS Raspbian Installer
- Format SD Card - FAT32 Single Partition
- Unzip NOOBS zip file
- Transfer contents to SD Card

Hook up the Raspberry Pi to your TV & keyboard, boot into Noobs

- Install Raspbian
- Reboot into raspi-config
- Change default password
- Disable boot to GUI
- Fix localization options: en-US UTF8, US Central Timezone
- Enable SSH server
- Reallocate GPU memory to the smallest amount (16)
- Overclock if you want to
- Setup the hostname: hc0000, hc0001, hc0002

Setup your environment
----------------------

Set up static ip's for each node. This can be done through your router's web panel. Each node is filtered to an IP by it's ethernet port's mac address. For our examples, we'll use the following:

 - hc0000 - 192.168.1.10
 - hc0001 - 192.168.1.11
 - hc0002 - 192.168.1.12

Most of the steps from here forward assume you will be administering your cluster from a Linux command line. 

Let's setup passwordless ssh from your computer to all 3 nodes. We'll need to copy the public key from your computer out to the cluster:

	cat .ssh/id_rsa.pub
	ssh pi@hc0000
	mkdir .ssh
	echo "${public-key}" >> .ssh/authorized_keys

Hadoop prerequisites & base install
-----------------------------------

For many of the steps below we can run a bash loop to hit all 3 boxes at once. Something simple, like the following.

	echo "hc0000
	hc0001
	hc0002" | while read line
	do
		echo $line
		ssh -n pi@$line "uptime"
		echo
	done

This will run the command `uptime` on all 3 nodes.

For the sake of clarity, I'll refrain from pasting the loop at each step but instead just the single commands.

Run all of these next steps on all of the nodes.

Update the system

`sudo apt-get update; sudo apt-get -y upgrade`
	
Install Java

`sudo apt-get -y install openjdk-7-jdk`
	
Setup 'hduser' User and 'hadoop' Group

`sudo addgroup hadoop`
`sudo adduser --ingroup hadoop hduser`	

Adding the user failed when trying over ssh, can't set password without an interactive tty.

`sudo adduser hduser sudo`					

This step had to be done as 'pi' or 'root' user, with sudo access.
	
*Everything from this point forward has to be run as user 'hduser'*

Setup SSH so the nodes can talk to themselves and eachother.

`ssh-keygen -t rsa -P ""`
`cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys`

Test that it's working

`ssh localhost`
	
Download Hadoop

`cd ~`
`wget http://apache.mirrors.tds.net/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz`
	
Install Hadoop to /usr/local
