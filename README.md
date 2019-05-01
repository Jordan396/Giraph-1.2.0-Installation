# Giraph 1.2.0 Installation on Google VM
## Overview
With the latest changes, the installation guide on [Giraph's official website](http://giraph.apache.org/) has become outdated. For new users who are keen to explore Giraph, simply following the website's guide may result in much frustration due to issues with mismatched dependencies. Furthermore, users who are not running on an *Ubuntu* operating system may find it difficult to get started.

I have adapted from the [official guide](http://giraph.apache.org/quick_start.html) to create this updated guide. In what follows, we shall create a VM instance on *Google Cloud Platform* that will run on an *Ubuntu* OS. Even if *Ubuntu* is not your native OS, using a VM instance will allow you to get along just fine. We will then deploy Giraph on a single-node, pseudo-distributed Hadoop cluster.

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
![alt text](https://github.com/Jordan396/Giraph-1.2.0-Installation/raw/master/src/images/VM_settings2.PNG "VM Settings")

As this instance is rather expensive, do ensure that you have sufficient credits in your account before clicking 'Create'. Once done, follow these [instructions](https://cloud.google.com/compute/docs/instances/connecting-to-instance) to connect to your instance.

## Deploying Hadoop
We will now deploy a single-node, pseudo-distributed Hadoop cluster. First, log into `root`, change the password and install vim and Java 1.8:
```
sudo -i
passwd
apt-get update
apt-get install vim
apt-get install openjdk-8-jdk
java -version
```
You should see your Java version information. After that, create a dedicated `hadoop` group, a new user account `hduser`, and then add this user account to the newly created group:
```
addgroup hadoop
adduser --ingroup hadoop hduser
```
Next, download and extract hadoop-2.5.1-src.tar.gz from [Apache archives](http://archive.apache.org/dist/hadoop/core/) (this is the most updated version compatible with Giraph):
```
cd /usr/local
wget http://archive.apache.org/dist/hadoop/core/hadoop-2.5.1/hadoop-2.5.1.tar.gz
tar xzf hadoop-2.5.1.tar.gz
mv hadoop-2.5.1 hadoop
chown -R hduser:hadoop hadoop
```
After installation, switch to user account `hduser` and edit the account's `$HOME/.bashrc`:
```
su - hduser
vim .bashrc
# Add the following line to the end of the bash file
export HADOOP_HOME=/usr/local/hadoop
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
# After exiting the visual editor, execute the bash script
source $HOME/.bashrc
```
This will set Hadoop/Java related environment variables. After that, edit `$HADOOP_HOME/etc/hadoop/hadoop-env.sh`:
```
cd $HADOOP_HOME/etc/hadoop/hadoop-env.sh
# Replace the export JAVA_HOME line with the following
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```
As Hadoop stores temporary files during its computation, you need to create a base temporary directory for local FS and HDFS files as follows:
```
su -
mkdir -p /app/hadoop/tmp
chown hduser:hadoop /app/hadoop/tmp
chmod 750 /app/hadoop/tmp
```
Navigate to `/etc/hosts` and add the following line:
```
cd /etc
vim hosts
# Add the following line below localhost
123.123.12.12   hdnode01
```
Even though we can use localhost for all communication within this single-node cluster, using the hostname is generally a better practice (e.g., you might add a new node and convert your single-node, pseudo-distributed cluster to multi-node, distributed cluster).

Now, edit Hadoop configuration files `core-site.xml`, `mapred-site.xml.template`, and `hdfs-site.xml` under `$HADOOP_HOME/etc/hadoop` to reflect the current setup. Add the new lines between `<configuration>...</configuration>`, as specified below:

* Edit `core-site.xml` with: 

```
<property>
<name>hadoop.tmp.dir</name>
<value>/app/hadoop/tmp</value>
</property>

<property> 
<name>fs.default.name</name> 
<value>hdfs://hdnode01:54310</value> 
</property>
```

* Edit `mapred-site.xml.template` with: 

```
<property>
<name>mapred.job.tracker</name> 
<value>hdnode01:54311</value>
</property>

<property>
<name>mapred.tasktracker.map.tasks.maximum</name>
<value>4</value>
</property>

<property>
<name>mapred.map.tasks</name>
<value>4</value>
</property>
```

By default, Hadoop allows 2 mappers to run at once. Giraph's code, however, assumes that we can run 4 mappers at the same time. Accordingly, for this single-node, pseudo-distributed deployment, we need to add the last two properties in `mapred-site.xml.template` to reflect this requirement. Otherwise, some of Giraph's unittests will fail.
 
* Edit `hdfs-site.xml` with: 

```
<property>
<name>dfs.replication</name> 
<value>1</value> 
</property>
```

Notice that you just set the replication service to make only 1 copy of the files stored in HDFS. This is because you have only one data node. The default value is 3 and you will receive run-time exceptions if you do not change it.
 
Next, set up SSH for user account `hduser` so that you do not have to enter a passcode every time an SSH connection is started:

```
su - hduser
ssh-keygen -t rsa -P ""
cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys
```

And then SSH to hdnode01 under user account `hduser` (this must be to hdnode01, as we used the node's hostname in Hadoop configuration). You will be asked for a password if this is the first time you SSH to the node under this user account. When prompted, do store the public RSA key into `$HOME/.ssh/known_hosts`. Once you make sure you can SSH without a passcode/password, edit `$HADOOP_HOME/etc/hadoop/slaves`:

```
cd $HADOOP_HOME/etc/hadoop/slaves
vim slaves
# Replace localhost with the following
hdnode01
```

These edits set a single-node, pseudo-distributed Hadoop cluster consisting of a single master and a single slave on the same physical machine. Note that if you want to deploy a multi-node, distributed Hadoop cluster, you should add other data nodes (e.g., `hdnode02`, `hdnode03`, ...) in the `$HADOOP_HOME/etc/hadoop/slaves` file after following all of the steps above on each new node with minor changes. 

Let us move on. To initialize HDFS, format it by running the following command:
```
$HADOOP_HOME/bin/hdfs namenode -format
```
And then start the HDFS daemons in the following order:
```
$HADOOP_HOME/sbin/start-all.sh
```
To stop the daemons, run the equivalent `$HADOOP_HOME/sbin/stop-all.sh` script. This is important so that you will not lose your data. You are done with deploying a single-node, pseudo-distributed Hadoop cluster.

## Deploying Giraph
We will now deploy Giraph. In order to build Giraph from the repository, you need first to install Git and Maven 3 by running the following commands:
```
su -
apt-get install git
apt-get install maven
mvn -version
```
Make sure that you have installed Maven 3 or higher. Giraph uses the Munge plugin, which requires Mave 3, to support multiple versions of Hadoop. Also, the web site plugin requires Maven 3. You can now clone Giraph from its Github mirror:
```
cd /usr/local/
git clone https://github.com/apache/giraph.git
chown -R hduser:hadoop giraph
su - hduser
```
After that, edit `$HOME/.bashrc` for user account hduser with the following line:
```
export GIRAPH_HOME=/usr/local/giraph
```
Save and close the file, and then validate, compile, test (if required), and then package Giraph into JAR files by running the following commands:
```
source $HOME/.bashrc
cd $GIRAPH_HOME
mvn package -DskipTests
```
The argument `-DskipTests` will skip the testing phase. This may take a while on the first run because Maven is downloading the most recent artifacts (plugin JARs and other files) into your local repository. You may also need to execute the command a couple of times before it succeeds. This is because the remote server may time out before your downloads are complete. 

## Running a Giraph job
With Giraph and Hadoop deployed, you can run your first Giraph job. We will use the *SimpleShortestPathsComputation* example job which reads an input file of a graph in one of the supported formats and computes the length of the shortest paths from a source node to all other nodes. The source node is always the first node in the input file. We will use *JsonLongDoubleFloatDoubleVertexInputFormat* input format. First, create an example graph under `/tmp/tiny_graph.txt`:
```
cd /tmp
vim tiny_graph.txt
# Paste the following lines into the text file
[0,0,[[1,1],[3,3]]]
[1,0,[[0,1],[2,2],[3,1]]]
[2,0,[[1,2],[4,4]]]
[3,0,[[0,3],[1,1],[4,4]]]
[4,0,[[3,4],[2,4]]]
```
Save and close the file. Each line above has the format `[source_id,source_value,[[dest_id, edge_value],...]]`. In this graph, there are 5 nodes and 12 directed edges. Copy the input file to HDFS:
```
$HADOOP_HOME/bin/hdfs dfs -mkdir /user
$HADOOP_HOME/bin/hdfs dfs -mkdir /user/hduser
$HADOOP_HOME/bin/hdfs dfs -mkdir /user/hduser/input
$HADOOP_HOME/bin/hdfs dfs -copyFromLocal /tmp/tiny_graph.txt /user/hduser/input/tiny_graph.txt
$HADOOP_HOME/bin/hdfs dfs -ls /user/hduser/input
```
We will use *IdWithValueTextOutputFormat* output file format, where each line consists of source_id length for each node in the input graph (the source node has a length of 0, by convention). You can now run the example by:
```
$HADOOP_HOME/bin/hadoop jar $GIRAPH_HOME/giraph-examples/target/giraph-examples-1.2.0-hadoop2-for-hadoop-2.5.1-jar-with-dependencies.jar org.apache.giraph.GiraphRunner org.apache.giraph.examples.SimpleShortestPathsComputation -vif org.apache.giraph.io.formats.JsonLongDoubleFloatDoubleVertexInputFormat -vip /user/hduser/input/tiny_graph.txt -vof org.apache.giraph.io.formats.IdWithValueTextOutputFormat -op /user/hduser/output/shortestpaths -w 1 -ca giraph.SplitMasterWorker=false
```
Notice that the job is computed using a single worker using the argument `-w`. To get more information about running a Giraph job, run the following command:
```
$HADOOP_HOME/bin/hadoop jar $GIRAPH_HOME/giraph-examples/target/giraph-examples-1.2.0-hadoop2-for-hadoop-2.5.1-jar-with-dependencies.jar org.apache.giraph.GiraphRunner -h
```
This will output the following:
```
usage: org.apache.giraph.utils.ConfigurationUtils [-aw <arg>] [-c <arg>]
       [-ca <arg>] [-cf <arg>] [-eif <arg>] [-eip <arg>] [-eof <arg>]
       [-esd <arg>] [-h] [-jyc <arg>] [-la] [-mc <arg>] [-op <arg>] [-pc
       <arg>] [-q] [-th <arg>] [-ve <arg>] [-vif <arg>] [-vip <arg>] [-vof
       <arg>] [-vsd <arg>] [-vvf <arg>] [-w <arg>] [-wc <arg>] [-yh <arg>]
       [-yj <arg>]
 -aw,--aggregatorWriter <arg>           AggregatorWriter class
 -c,--messageCombiner <arg>             Message messageCombiner class
 -ca,--customArguments <arg>            provide custom arguments for the
                                        job configuration in the form: -ca
                                        <param1>=<value1>,<param2>=<value2>
                                        -ca <param3>=<value3> etc. It
                                        can appear multiple times, and the
                                        last one has effect for the sameparam.
 -cf,--cacheFile <arg>                  Files for distributed cache
 -eif,--edgeInputFormat <arg>           Edge input format
 -eip,--edgeInputPath <arg>             Edge input path
 -eof,--vertexOutputFormat <arg>               Edge output format
 -esd,--edgeSubDir <arg>                subdirectory to be used for the
                                        edge output
 -h,--help                              Help
 -jyc,--jythonClass <arg>               Jython class name, used if
                                        computation passed in is a python
                                        script
 -la,--listAlgorithms                   List supported algorithms
 -mc,--masterCompute <arg>              MasterCompute class
 -op,--outputPath <arg>                 Vertex output path
 -pc,--partitionClass <arg>             Partition class
 -q,--quiet                             Quiet output
 -th,--typesHolder <arg>                Class that holds types. Needed
                                        only if Computation is not set
 -ve,--outEdges <arg>                   Vertex edges class
 -vif,--vertexInputFormat <arg>         Vertex input format
 -vip,--vertexInputPath <arg>           Vertex input path
 -vof,--vertexOutputFormat <arg>        Vertex output format
 -vsd,--vertexSubDir <arg>              subdirectory to be used for the
                                        vertex output
 -vvf,--vertexValueFactoryClass <arg>   Vertex value factory class
 -w,--workers <arg>                     Number of workers
 -wc,--workerContext <arg>              WorkerContext class
 -yh,--yarnheap <arg>                   Heap size, in MB, for each Giraph
                                        task (YARN only.) Defaults to
                                        giraph.yarn.task.heap.mb => 1024
                                        (integer) MB.
 -yj,--yarnjars <arg>                   comma-separated list of JAR
                                        filenames to distribute to Giraph
                                        tasks and ApplicationMaster. YARN
                                        only. Search order: CLASSPATH,
                                        HADOOP_HOME, user current dir.
```
**NOTE:** When setting up the VM instance earlier, notice that we did not allow all HTTP/HTTPS traffic due to security reasons. Here, if you would like to monitor the progress of your job and other cluster info using the web UI, you will have to open ports `50070`, `50060` and `50030`.
* NameNode daemon: http://34.345.345.345:50070
* JobTracker daemon: http://34.345.345.345:50030
* TaskTracker daemon: http://34.345.345.345:50060


Once the job is completed, you can check the output by running:
```
$HADOOP_HOME/bin/hdfs dfs -text /user/hduser/output/shortestpaths/p* | less
```
That's all I have for now. Do let me know if you encounter any issues with the above instructions by dropping me an email at [jordan.chyehong@gmail.com](mailto:jordan.chyehong@gmail.com). 
