---
title: "Setting up Giraph on a Hadoop cluster"
layout: post
description: A guide to setting up Giraph 1.2.0
author: anirudh
blog: true
tag:
- hadoop
- giraph
headerImage: false
---

## Introduction

In the [previous post]({% post_url 2016-09-19-hadoop-setup %}) we discussed how to install Hadoop on a cluster. Now there a dozen guides available to help you with that, so that, of course, was not my primary purpose. Building up from what we achieved in the previous post, here we talk about how to setup Giraph on a Hadoop cluster, something you would find rather helpful.

We would be setting up Giraph v1.2 to run Giraph tasks as run in YARN-managed execution
containers. Let's get started.

## Setting up Giraph

### Fetching Giraph distribution

We would be fetching the Giraph source from GitHub and building it from the source code. Execute the following command to clone the giraph repository into your current directory.

{% highlight shell %}
git clone https://github.com/apache/giraph.git
{% endhighlight %}

### Switching to release-1.2 branch

Now that we have cloned the Giraph repository, let's switch to the commit having code for the version of Giraph we are interested in. Run the following commands to do that.

{% highlight shell %}
cd giraph
git checkout release-1.2
{% endhighlight %}

### Modifying pom.xml

We would be building Giraph for Hadoop v2.7.2 with YARN support. Trying to compile the source code right away would result  in a compilation error stating that `SASL_PROPS` symbol could not be found. To get around this we would need to remove the `STATIC_SASL_SYMBOL` munge symbol under hadoop_yarn profile in `pom.xml`. Open your favourite text editor and edit the line `<munge.symbols>PURE_YARN,STATIC_SASL_SYMBOL</munge.symbols>` to the following

{% highlight xml %}
<munge.symbols>PURE_YARN</munge.symbols>
{% endhighlight %}

Under the `hadoop_yarn` profile, add the version tags to the `hadoop-common, hadoop-mapreduce-client-common, hadoop-mapreduce-client-core` dependencies.

{% highlight xml %}
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>${hadoop.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-mapreduce-client-common</artifactId>
    <version>${hadoop.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-mapreduce-client-core</artifactId>
    <version>${hadoop.version}</version>
</dependency>
{% endhighlight %}



### Building Giraph from source

We use Maven to build Giraph with the following command, specifying the profile `hadoop_yarn` and hadoop version `2.7.2` as command line parameters.

{% highlight shell %}
mvn –Phadoop_yarn –Dhadoop.version=2.7.2 -DskipTests clean package
{% endhighlight %}

## Wrapping it up

The Maven package phase generates jars for each Giraph module. These jars can be added as dependencies in your projects, and you can run your code using Hadoop after exporting it as a jar. For an example of running a Giraph task as a Hadoop job, have a look at the [official quick start guide](http://giraph.apache.org/quick_start.html#qs_section_6).



