---
title: "Setting up Hadoop on a cluster"
layout: post
description: A guide to setting up Hadoop 2.7.2
author: anirudh
blog: true
tag:
- hadoop
headerImage: false
---

## Introduction

In this post I would covering the steps to setup a Hadoop multi-node cluster. By the end of the post you will have your cluster up and running for accepting Hadoop jobs.

The setup was tested on machines running Ubuntu 16.04, and Hadoop v2.7.2.

## Setting up your machines

Do the following on each of your machines to get them ready to run Hadoop.

### Modifying hosts file

Fetch the IP addresses of all the machines in your cluster, and add the addresses to the /etc/hosts file accordingly. Designate one machine as the master and the rest as slaves, name them accordingly.

{% highlight text %}
master 10.0.0.2
slave1 10.0.0.3
slave2 10.0.0.4
slave3 10.0.0.5
{% endhighlight %}


### Passphraseless SSH

To get Hadoop to work you would need to be able to ssh to the localhost without a passphrase. Execute the following commands to enable it.

{% highlight shell %}
ssh-keygen -t rsa -P ""
cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys
{% endhighlight %}

In addition, your master should be able to connect to the slave machines without a passphrase. Run the following command <span class="evidence">on the master</span>, for each slave. Replace username with the `username` for the account on the slave machine you intend to setup up Hadoop on, and `X` by the number for each of the slave as mentioned the etc/hosts file.

{% highlight shell %}
ssh-copy-id -i ~/.ssh/id_rsa.pub username@slaveX
{% endhighlight %}

### Installing OpenJDK

{% highlight shell %}
sudo apt-get install openjdk-8-jdk
{% endhighlight %}

## Setting up Hadoop

Do the following on your master node. We would copy the configuration to each of the slaves, at the end, with a nifty piece of code.

### Fetching Hadoop distribution

We would be setting up everything in the home directory for each of the machines in the cluster. Run the following commands on your master node, to fetch Hadoop v2.7.2 distribution, extract it, and put it in a folder called hadoop.

{% highlight shell %}
wget http://mirror.fibergrid.in/apache/hadoop/common/hadoop-2.7.2/hadoop-2.7.2.tar.gz
tar xvf hadoop-2.7.2.tar.gz
mv hadoop-2.7.2 hadoop
{% endhighlight %}

### Editing hadoop-env.sh

The `hadoop-env.sh` file should be located at `~/hadoop/etc/hadoop/hadoop-env.sh`. Open it in your favourite editor and edit the line `export JAVA_HOME=$(JAVA_HOME)` to the following

{% highlight shell %}   
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
{% endhighlight %}

### Editing core-site.xml

The `core-site.sh` file should be located at `~/hadoop/etc/hadoop/core-site.sh`. Open it in your favourite editor and add the `fs.defaultFS` property under inside the configuration tag, like so

{% highlight xml %}
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
{% endhighlight %}

### Editing hdfs-site.xml

The `hdfs-site.sh` file should be located at `~/hadoop/etc/hadoop/hdfs-site.sh`. Add the dfs.replication property to define the number of machines a file should be replicated to when being stored in HDFS. If using a single slave node, set it to 1, if using 2, set it to 2, for 3 or more slaves use the default replication of 3.

{% highlight xml %}
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
{% endhighlight %}

### Editing yarn-site.xml

The `yarn-site.sh` file should be located at `~/hadoop/etc/hadoop/yarn-site.sh`. Edit the yarn.resourcemanager.hostname property and set its value to `master`, the DNS entry in the hosts file for the master.

{% highlight xml %}
<configuration>
 <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>master</value>
  </property>
</configuration>
{% endhighlight %}

### Editing slaves file

The `slaves` file should be located at `~/hadoop/etc/hadoop/slaves`. Replace the contents of the file with the DNS entries for your slaves as mentioned in `/etc/hosts`.

{% highlight text %}
slave1
slave2
slave3
{% endhighlight %}

### Configuring environment variables

Edit your ~/.bashrc file, add the following lines at the end to setup up the environment variables needed by Hadoop. Do this on <span class="evidence">every machine</span> in the cluster.

{% highlight shell %}
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_PREFIX=~/hadoop
export PATH=$PATH:$HADOOP_PREFIX/bin
{% endhighlight %}

### Copy hadoop directory to slaves

Now that we have all the configuration in place, we can copy the `hadoop` directory each of the slaves with the following command. Replace username with the `username` for the account on the slave machine you intend to setup up Hadoop on, and `X` by the number for each of the slave as mentioned the etc/hosts file.

{% highlight shell %}
scp -r ~/hadoop/ username@slaveX:~
{% endhighlight %}

### Formatting the NameNode

Before you start using HDFS, the NameNode, which contains the directory structure for the files stored in HDFS needs to be formatted. Use the following command to do it.

{% highlight shell %}
hdfs namenode -format
{% endhighlight %}

## Get it up and running

Run the following commands on your master node to start up HDFS and yarn

{% highlight shell %}
~/hadoop/sbin/start-dfs.sh
~/hadoop/sbin/start-yarn.sh
{% endhighlight %}
