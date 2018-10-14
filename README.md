# Giraph-1.2.0-Installation on Google VM
## Overview
With the latest changes in [Giraph](http://giraph.apache.org/), the installation guide on Giraph's official website has become rather outdated. For new users who are keen to explore Giraph, simply following the official guide may result in much frustration due to issues owing to mismatched dependencies. Furthermore, users who are not running on an *Ubuntu* system may find it difficult to get started.

Consequently, I have adapted from the [official guide](http://giraph.apache.org/quick_start.html) to create this updated guide which addresses the above issues. In what follows, we shall begin by creating a VM instance on *Google Cloud Platform* that will run on an *Ubuntu* operating system. We will then deploy a single-node, pseudo-distributed Hadoop cluster. We will also deploy Giraph on this node. 

The deployment uses the following software/configuration:

 * Ubuntu Server 18.04 LTS with the following configuration:
  * 4 vCPUs with 15 GB memory
  * 10 GB standard persistent disk 
  * Admin account: `root`
  * Hostname: `hdnode01`
  * Internal IP address: `123.123.12.12`
  * External IP address: `34.345.345.345`
 * Apache Hadoop 2.5.1
 * Apache Giraph 1.2.0

## Creating a VM Instance
For detailed instructions, please refer to this [link](https://cloud.google.com/compute/docs/instances/create-start-instance). Create the instance according to these settings.


