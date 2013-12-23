configure-hadoop-raspi
==============

Configuring a Hadoop Cluster with Raspberry Pi's

Welcome!
--------

To be clear, the following setup is impractical and should be done purely as a learning excercise. The JVM (Java Virtual Machine) which powers Hadoop is horridly slow on the ARM CPUs that drive the Raspberry Pi. Feel free to use any model Raspberry Pi as none of them will be able to do any timely data crunching anyways. This project exists solely to help in booting up your first Hadoop Environment! 

Technically you could draw more computing power out of the setup with a few of the following considerations, but that's beyond the scope of this guide: Hard Float OS, Oracle JDK, JVM tuning params.

Before we start
---------------

The following items are not all necessary. Obviously, you could have any combination of equipment. Technically you could run all of the hadoop tools on a single node if you wanted. By choosing these items we're just laying out the most common path we assume people will want to take. If anything below is specific to these items, we'll point it out.

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

For the rest of this section we can run a bash loop to hit all 3 boxes at once. Something simple, like the following.

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

Update the system

	sudo apt-get update; sudo apt-get -y upgrade
	
Install Java

	sudo apt-get -y install openjdk-7-jdk
	
Setup 'hduser' User and 'hadoop' Group

	sudo addgroup hadoop
	sudo adduser --ingroup hadoop hduser

Adding the user may fail when trying to do so over ssh. This is because you can't set the password without an interactive shell.

Here we'll give `hduser` sudo access. This step must be done as `root` user or someone with sudo access.

	sudo adduser hduser sudo

*WARNING: Everything from this point forward has to be run as user 'hduser'*

Setup SSH so the nodes can talk to themselves and eachother

	ssh-keygen -t rsa -P ""
	cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

Test that it's working

`ssh localhost`
	
Download Hadoop

	cd ~
	wget http://apache.mirrors.tds.net/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz
	
Install Hadoop to /usr/local

	sudo tar vxzf hadoop-1.2.1.tar.gz -C /usr/local/
	cd /usr/local
	sudo mv hadoop-1.2.1/ hadoop
	sudo chown -R hduser:hadoop /usr/local/hadoop
	
Configure User Environment

	cd ~
	rm hadoop-1.2.1.tar.gz

You can use any editor you like but I'll denote the moments when we're supposed to edit a text file with vim.

`vim .bash_profile`

Add the following to each node's `~/.bash_profile`

	# Setup Environment Variables
	export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-armhf
	export HADOOP_INSTALL=/usr/local/hadoop
	export PATH=$PATH:$HADOOP_INSTALL/bin

Setup the `hadoop-env.sh`	

`vim /usr/local/hadoop/conf/hadoop-env.sh`
	
The options to adjust here are:

	export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-armhf
	export HADOOP_HEAPSIZE=384	

Some other similar guides had set `HADOOP_HEAPSIZE` to 272. I raised it up a bit because there was 433 MB free upon issuing a `free -m` after cold boot.

Setup the master node
---------------------

Setup the tmp dir and filesystem URI

`vim /usr/local/hadoop/conf/core-site.xml`

Add the following to the <configuration> section:

	<property>
	  <name>hadoop.tmp.dir</name>
	  <value>/fs/hadoop/tmp</value>
	  <description>Sets the operating directory for Hadoop data.
	  </description>
	</property>
	<property>
	  <name>fs.default.name</name>
	  <value>hdfs://localhost:54310</value>
	  <description>The name of the default file system.  A URI whose
	  scheme and authority determine the FileSystem implementation.
	  The URI's scheme determines the config property (fs.SCHEME.impl) naming
	  the FileSystem implementation class.  The URI's authority is used to
	  determine the host, port, etc. for a filesystem.  
	  </description>
	</property>

Set up the job tracker URI

`vim /usr/local/hadoop/conf/mapred-site.xml`

Add the following to the <configuration> section:

	<property>
	  <name>mapred.job.tracker</name>
	  <value>localhost:54311</value>
	  <description>The host and port that the MapReduce job tracker runs
	  at.  If "local", then jobs are run in-process as a single map
	  and reduce task.
	  </description>
	</property>

Set the replication factor to 1 for now, since we only have 1 node.

`vim /usr/local/hadoop/conf/hdfs-site.xml`

Add the following to the <configuration> section:

	<property>
	  <name>dfs.replication</name>
	  <value>1</value>
	  <description>Default block replication.
	  The actual number of replications can be specified when the file is created.
	  The default is used if replication is not specified in create time.
	  </description>
	</property>

Configure the hdfs on real disk

	sudo mkdir -p /fs/hadoop/tmp
	sudo chown hduser:hadoop /fs/hadoop/tmp
	sudo chmod 750 /fs/hadoop/tmp/

Format the HDFS

	/usr/local/hadoop/bin/hadoop namenode -format

Start it up!

	/usr/local/hadoop/bin/start-all.sh

You can run 'jps' command to ensure all the appropriate Java Hadoop processes are running:

	DataNode
	TaskTracker
	SecondaryNameNode
	NameNode
	JobTracker

If you've encountered failures look at the logs:

	/usr/local/hadoop/logs

Logs are titled like:

	hadoop-(username)-(processtype)-(machinename).log

If everything is working as expected you should be able to hit the following pages:

- NameNode - 192.168.1.10:50070
- JobTracker - 192.168.1.10:50030
- TaskTracker - 192.168.1.10:50060

Now that everything is up and running we'll run our first job, a word count example!

Download any book in "Plain Text UTF-8" format from www.gutenburg.org and load it into HDFS:

	mkdir -p /tmp/books
	cd /tmp/books
	wget http://www.gutenberg.org/files/43790/43790-0.txt
	mv 43790-0.txt bookofcats.txt
	/usr/local/hadoop/bin/hadoop dfs -copyFromLocal /tmp/books /fs/hduser/books

Run the wordcount example:

	/usr/local/hadoop/bin/hadoop jar /usr/local/hadoop/hadoop*examples*.jar wordcount /fs/hduser/books /fs/hduser/books-output

Setup the slave nodes
---------------------

Now, let's work with multiple nodes.

Master node, hc0000, will run:

	NameNode
	Secondary NameNode
	DataNode
	JobTracker (In a large production cluster this would be somewhere else)
	TaskTracker

In a large production cluster Secondary NameNode and JobTracker would have their own servers.

Slave nodes, hc0001 and hc0002, will run:

	DataNode
	TaskTracker

Before we begin again, setup passwordless ssh between all nodes:

Gather up all keys with a loop

	cat ~/.ssh/id_rsa.pub

Take these public keys and add them to the authorized_keys file of each node with a loop
	
	echo "${public_keys}" >> ~/.ssh/authorized_keys


On the master, hc0000
---------------------

These two files, masters and slaves, control the way the cluster stops and starts. You can remove hosts from here and reboot the cluster to kick a node out, or conversely bring one in.

`vim /usr/local/hadoop/conf/masters`

	Change:
		localhost
	To:
		hc0000

`vim /usr/local/hadoop/conf/slaves`

	Change:
		localhost
	To:
		hc0000
		hc0001
		hc0002

`vim /usr/local/hadoop/conf/core-site.xml`

	Change:
		<name>fs.default.name</name>
		<value>hdfs://localhost:54310</value>

	To reflect the hostname of the server running NameNode:
		<name>fs.default.name</name>
		<value>hdfs://hc0000:54310</value>

`vim /usr/local/hadoop/conf/mapred-site.xml`

	Change:
		<name>mapred.job.tracker</name>
		<value>localhost:54311</value>

	To:
		<name>mapred.job.tracker</name>
		<value>hc0000:54311</value>

`vim /usr/local/hadoop/conf/hdfs-site.xml`

	Change:
		<name>dfs.replication</name>
		<value>1</value>

	To:
		<name>dfs.replication</name>
		<value>3</value>

Copy core-site.xml, mapred-site.xml and hdfs-site.xml out from the master to the new nodes:

	scp hduser@hc0000:/usr/local/hadoop/conf/core-site.xml hduser@hc0001:/usr/local/hadoop/conf/.
	scp hduser@hc0000:/usr/local/hadoop/conf/mapred-site.xml hduser@hc0001:/usr/local/hadoop/conf/.
	scp hduser@hc0000:/usr/local/hadoop/conf/hdfs-site.xml hduser@hc0001:/usr/local/hadoop/conf/.

	scp hduser@hc0000:/usr/local/hadoop/conf/core-site.xml hduser@hc0002:/usr/local/hadoop/conf/.
	scp hduser@hc0000:/usr/local/hadoop/conf/mapred-site.xml hduser@hc0002:/usr/local/hadoop/conf/.
	scp hduser@hc0000:/usr/local/hadoop/conf/hdfs-site.xml hduser@hc0002:/usr/local/hadoop/conf/.

On the slaves, hc0001 and hc0002
--------------------------------

Configure the hdfs on real disk

	sudo mkdir -p /fs/hadoop/tmp
	sudo chown hduser:hadoop /fs/hadoop/tmp
	sudo chmod 750 /fs/hadoop/tmp/

On all nodes
------------

sudo vi /etc/hosts

	Change:
		127.0.1.1       hc0000
	To:
		192.168.1.10	hc0000
		192.168.1.11	hc0001
		192.168.1.12	hc0002

Leave the rest of the file intact.

Reboot each node to make sure these settings update

	sudo reboot

On the master
-------------

Format the HDFS

	/usr/local/hadoop/bin/hadoop namenode -format

Start it up

	/usr/local/hadoop/bin/start-all.sh

Congratulations!
----------------

You can run 'jps' again to make sure all the right processes are up and running.

You should now be able to view the following web pages:

NameNode

	192.168.1.10:50070

JobTracker

	192.168.1.10:50030

TaskTrackers

	192.168.1.10:50060
	192.168.1.11:50060
	192.168.1.12:50060

Congratulations on setting up your first Hadoop Environment!

Sources
-------

This blog post was an invaluable resource in setting up this cluster:
http://blog.ittoby.com/2013/08/starting-small-set-up-hadoop-compute.html
