---
layout: post
title: "Open Source Java client for Cloudera Manager API"
description: "Automate Hadoop deployment"
category: 
tags: [automation]
---
{% include JB/setup %}

# Automate Hadoop deployment for testing 

##Motivation 

Here at <a href="http://axembrl.com/">Axembrl</a> we have a great desire: to be Number 1 at delivering <a href="http://hadoop.apache.org/">Hadoop</a> on Cloud infrastructure. And because of that, things like packaging, installing and configuring Hadoop are not our primary focus. We still need them but hey, let somebody else do it. Especially if that somebody else does a great job, like the people from <a href="http://cloudera.com">Cloudera</a> do. 

In the pursuit of our greatest desire (second only to coffee early in the morning) we ended up writing a Java client for Cloudera Manager's API. Thus we achieved to automate a CDH3 Hadoop installation on Amazon EC2 and Rackspace Clood. We also decided to open source the client so other people can play along. Of course such an achievement also requires us to brag about it so I hope you will enjoy and learn from my short presentation of hot to deploy Hadoop on cloud infrastructure with Cloudera Manager!

###The problem 

So we need to deploy Hadoop on Cloud with as little user interaction as possible. We have the code to provision the hosts but we still need to install and configure Hadoop on all nodes and make it so the user has a nice experience doing it. And just to as a note, one of the things I learned is that complex distributed systems like Hadoop and friends are not that easy to set-up automatically. Not easy for everybody else, that is :D. 

###Cloudera Manger to the rescue

Fortunately the guys from Cloudera, who package Hadoop also have a product called <a href="http://www.cloudera.com/products-services/tools/">Cloudera Manger</a> (CM for short). Cloudera Manager facilitates the installation and configuration of Hadoop clusters. Fortunately for us Cloudera Manager version 4 added a REST API that's very nicely documented <a href="http://cloudera.github.com/cm_api/">here</a>, but also lacks some features (node discovery/configuration) exposed in web interface. Cloudera Manager comes in two flavors: Free and Enterprise Edition. We are going to use the free edition which can install only one cluster of maximum 50 nodes, but that's way too much for our needs.

###Minimum requirements

For the sake of simplicity I'm going to present only how to use the API to automate Hadoop installation on a single virtual machine. We will make use of <a href="http://www.virtualbox.org/">VirtualBox</a> as a nice, easy to use virtualization solution and <a href="http://vagrantup.com/">Vagrant</a>. Vagrant is a tool built around VirtualBox and allows us to create reproducible and portable virtual environments. We will be able to automate our set-up and speed up testing. Furthermore we tested the set-up Ubuntu and MacOS X.

Our purpose is to create a VM with Cloudera Manager and CDH3 pre-installed as Cloudera Manager is not yet able to install CDH via API :-(. We are going to package this as a <a href="">Vagrant Box</a> that we will then use as a base for our set-up. All vm's will be created from this base machine. 
 
**Checklist**

 * a machine with > 6GB of RAM 
 * fast Internet connection
 * a development stack with git, Java, maven, VirtualBox and Vagrant
 * project sources

### Sources

First get the <a href="https://github.com/axemblr/cloudera-manager-api">sources</a>, and export an environment variable for easy reference: 

{% highlight bash %}

$ git clone git://github.com/axemblr/cloudera-manager-api.git
$ cd cloudera-manager-api && export CLIENT=`pwd`

{% endhighlight %}

### Making our base VM 

This should be quite simple because I have prepared a script that does all the work. The next line will create a new virtual machine and install Hadoop and Cloudera Manager on it.

{% highlight bash %}

$ cd $CLIENT/src/test/resources/vagrant/build-image && vagrant up     

{% endhighlight %}

Vagrant will first download a Ubuntu Lucid amd64 *box*, named **lucid64** from <a href="http://files.vagrantup.com/lucid64.box">Vagrant website</a> if it does not yet exist. After this it will run a provisioning script to install Cloudera Manger Server, Agent and also Cloudera CDH3 because we can't do that via API. The script is right besides the *Vagrant* configuration file and is called **install-cm-and-cdh.sh**

We should have a VM with all the goodies installed. You could check if Cloudera Manager is working by logging in the machine and checking port 7800.

{% highlight bash %}

$ vagrant ssh
$ vagrant@lucid64$ netstat -tupan | grep 7180

{% endhighlight %}

Now that things work let's pack this box and register it as a vagrant template so we can create as many identical machines we like. 

{% highlight bash %}
$ vagrant package 
$ vagrant box add lucid64-with-cm4 package.box
{% endhighlight %}

This should complete the set-up. Now we have a vagrant box called **lucid64-with-cm4** and we can start as many VM's as our RAM holds. 

We have a VM with Hadoop installed and now we have to configure and start a cluster. It's just a one node cluster, but hey, the principles are the same. 

We will configure and start our cluster from Java code so we need to have network access to the machine. We will configuring Vagrant with <a href="http://vagrantup.com/v1/docs/host_only_networking.html">Host-Only Networking</a>. We also need to give our future VM enough memory, 2GB should be enough. 

{% highlight bash %}
$ cd $CLIENT/src/test/resources/vagrant/
{% endhighlight %}

In this directory you can see the final Vagrant configuration file that looks something like this:

{% highlight ruby %}
Vagrant::Config.run do |config|
  # Note: this is the box we just created in 'Making the base VM'
  config.vm.box = "lucid64-with-cm4"

  # give it some ram
  config.vm.customize ["modifyvm", :id, "--memory", 2048]

  # hack to avoid 'vagrant up'  hang
  config.vm.customize ["modifyvm", :id, "--rtcuseutc", "on"]


  # Assign this VM to a host-only network IP, allowing you to access it
  # via the IP. Host-only networks can talk to the host machine as well as
  # any other machines on the same network, but cannot be accessed (through this
  # network interface) by any external networks.
  config.vm.network :hostonly, "192.168.33.10"

end
{% endhighlight %}

**Final Test**

If you run *vagrant up* in this directory you should get a machine with Hadoop installed that also runs Cloudera Manager. You can open a browser to <a href="http://192.168.33.10:7180">http://192.168.33.10:7180</a> and login with *admin*:*admin*

### Setting up the cluster - test set-up

Status: we are able to start a VM that has Hadoop installed and also runs Cloudera Manager and we are able to connect over network to the machine. We can use Cloudera Manager web interface to create our cluster but where's the fun in that!?

We are going to write a (junit) test that does the following: 

* starts the vm instance we created and waits for it 
* configures and starts a cluster with a HDFS service
* stops the VM

Since we need the VM before we run our tests we will put this in a *@BeforeClass* method.

{% highlight java %}
@BeforeClass
public static void setUpResources() throws Exception {
//start the VM
processBuilder.directory(new File(VAGRANT_CONFIG_PATH).getCanonicalFile());
runAndWait(processBuilder, 5, VAGRANT_CMD, "up");
TimeUnit.SECONDS.sleep(30); // wait until Cloudera Manager Service starts
CLIENT = ClouderaManagerClient.withConnectionURI(ENDPOINT)
        .withAuth("admin", "admin").build();
CLIENT.enableHttpLogging(true);
}
{% endhighlight %}

In the above code we make use of a *<a href="http://docs.oracle.com/javase/7/docs/api/java/lang/ProcessBuilder.html">java.lang.ProcessBuilder</a>* to call *vagrant up* as a separate process and wait for the command to finish.
Waiting for the VM to start is a bit tricky since services will not be available right away.
To keep it simple we will just sleep on it. We add another 30 second sleep for that.

The helper function *runAndWait* takes care of starting the process and waiting for it to finis. It cleans up the streams which is an <a href="http://kylecartmell.com/?p=9">important task</a> and also kills it if it's not done after a specified amount of time. 

{% highlight java %}

private static void runAndWait(ProcessBuilder pBuilder, 
    long minutesToWait,  String... commands) throws IOException {

pBuilder.command(commands);
pBuilder.redirectErrorStream(true);
Timer timer = null;
Process process = null;
  try {
    final Thread currentThread = Thread.currentThread();
    timer = new Timer(true);
    // interrupt the current thread so we avoid waiting indefinitely 
    timer.schedule(new TimerTask() {
      @Override
      public void run() {
        currentThread.interrupt();
      }
    }, TimeUnit.MINUTES.toMillis(minutesToWait));
    process = pBuilder.start();
    EXECUTOR.execute(new StreamGobbler(process.getInputStream(), "INPUT"));
    process.waitFor();
  } catch (InterruptedException interruptedException) {
    LOGGER.severe("Timeout exceeded. Killing the process as it probably hanged.");
    process.destroy();
  } catch (Exception e) {
    LOGGER.severe("Exception starting vagrant VM");
    LOGGER.severe(e.getMessage());
  } finally {
    // If the process returns within the timeout period, we have to stop the 
    // interrupter so that it does not unexpectedly interrupt some other code 
    // later.
    timer.cancel();
    // We need to clear the interrupt flag on the current thread just in case
    // interrupter executed after waitFor had already returned but before timer.
    // cancel took effect.
    //
    // Oh, and there's also Sun bug 6420270 to worry about here.
    Thread.interrupted();
  }
}

{% endhighlight %}

We want our tests to be ignored and not fail if something bad happened and the VM or services have not started so we create a *@Before* method that uses jUnit *asumeTrue* to check for a necessary condition. If it's met we run the test, otherwise junit will ignore it. 

{% highlight java %}
@Before
public void setUp() throws Exception {
  // this will ignore tests if the endpoint is not accessible
  assumeTrue(isEndpointActive(ENDPOINT));
}
{% endhighlight %}

### Using Cloudera Manager Java API

All interactions with CM will be done through an instance of class *ClouderaManagerClient*, the *Client* from now on. The *Client* uses <a href="http://jersey.java.net/">Jersey REST Client</a> under the hood to do it's dirty work.

**Creating the client**

{% highlight java %}

final URI ENDPOINT = URI.create(String.format("http://%s:%d", CM_SERVICE_IP,7180));

ClouderaManagerClient CLIENT = ClouderaManagerClient.withConnectionURI(ENDPOINT)
    .withAuth("admin", "admin").build();

{% endhighlight %}

Now we can use the client to make HTTP requests to the server and drive our cluster. Before we dive in, a few words about the Client:

* it has a fluent API interface so you can chain methods
* returns immutable objects so it's safe to work with 
* tries to map Cloudera Manager API URL paths: to access */hosts* you do *CLIENT.hosts()* and get an object that exposes that API. 
* has extensive JavaDoc with links to the API documentation
* easy to extend, just override the methods from *ClouderaManagerClient*
* on failure, requests return UniformInterfaceException that you can query for information
* it has no support for monitoring yet, as they are not available in Cloudera Manager Free edition

Creating a cluster with Cloudera Manager usually goes like this: You first create an in memory model of the future cluster by creating services (HDFS, MapReduce, etc.) and assigning Roles to Hosts (DataNode, NameNode, etc.). When you start the service CM will start to do it's job. 

**Adding Hosts**

The first thing we need to do is register hosts that CM will use as cluster nodes. This is the host discovery step from the CM Web interface except that it does not do node installation (does not install CM agent or CDH). This is why we had to do it by hand. 

{% highlight java %}

HostList hosts = null;
try {
     hosts = CLIENT.hosts().createHosts(Sets.newHashSet(
     new Host("192.168.33.10", "lucid64")));
} catch (UniformInterfaceException e) { 
  LOGGER.info(e.getResponse().getEntity(ErrorMessage.class).toString());
}
  
{% endhighlight %}

**Create a cluster**

The following code will create a cluster named *cluster-01* . A cluster is just an empty shell, a name that will hold the services like HDFS and MapReduce. We will add those right away.

{% highlight java %}

Cluster cluster = new Cluster("cluster-01", ClusterVersion.CDH3);
ClusterList created = CLIENT.clusters().createClusters(Sets.newHashSet(cluster));

{% endhighlight%}

**Create HDFS service and Roles**

We create a new HDFS service called *hdfs-01* under our cluster and add roles to machines.
A HDFS service must have a NameNode, SecondaryNameNode and some DataNode's to operate.

Server requests return a representation of the model stored. For example creating a service 
return a *ServiceList* object that has information about the service. Most requests or responses will require a list or set of objects. Use a list with one element if you don't need more.

{% highlight java %}

// create a HDFS Service
ServiceSetupInfo hdfsSetup = new ServiceSetupInfo("hdfs-01", ServiceType.HDFS);
ServiceList serviceList = CLIENT.clusters().getCluster("cluster-01")
        .registerServices(new ServiceSetupList(jdfsSetup));

// assign HDFS roles to machines
Role nameNodeSetup = new Role("hdfs-nn", RoleType.NAMENODE, new HostRef("lucd64"));
Role secondaryNameNodeSetup = new Role("hdfs-secondary", RoleType.SECONDARYNAMENODE,
 new HostRef("lucid64"));
Role dataNodeSetup = new Role("hdfs-data1", RoleType.DATANODE, 
new HostRef("lucid64"));

Set<Role> roleSet = Sets.newHashSet(nameNodeSetup, secondaryNameNodeSetup, dataNodeSetup);

RoleList createdRoles = CLIENT.clusters().getCluster("cluster-01")
    .getService("hdfs-01").createRoles(new RoleList(roleSet));
    
{%endhighlight%}

**Configure HDFS service**

Before we go on and start the service we must configure it: set a data directory for the NameNode, etc. Actions usually return a list of commands and you can block until all of them are finished. 

For commands that don't finish right away use *waitFor* method to block until they do. 
One of these commands is *hdfsFormat*, an operation required before starting our service.

{% highlight java %}

// keep the service API resources for easy access 
ServiceApi hdfsApi = CLIENT.clusters().getCluster("cluster-01")
                                .getService("hdfs-01");
Config nameNodeDataDir = new Config(DFS_NAME_DIR_LIST, "/tmp/nn");
ConfigList configList = hdfsApi.updateRoleConfig("hdfs-nn", 
                                new ConfigList(nameNodeDataDir));

Config hdfsCheckPointDir = new Config(FS_CHECKPOINT_DIR_LIST, "/tmp/checkpoints");

configList = hdfsApi.updateRoleConfig("hdfs-secondary", 
                                new ConfigList(hdfsCheckPointDir));

Config datanodeDataDir = new Config(DFS_DATA_DIR_LIST, "/tmp/datanode");
configList = hdfsApi.updateRoleConfig("hdfs-data1", 
                                new ConfigList(datanodeDataDir));

BulkCommandList commands = hdfsApi.hdfsFormat(new RoleNameList("hdfs-nn"));
// wait for the commands to finish        
List<Command> result = CLIENT.commands().waitFor(commands);

{% endhighlight %}

**Start the HDFS service**

Starting the service is a one liner and like the rest of the commands returns immediately but it takes a while until the command finishes it's execution. Because other services depend on a HDFS service we will have to wait until it's started. 

{% highlight java%}

// start the service
Command command = CLIENT.clusters().getCluster(CLUSTER_NAME)
                            .getService("hdfs-1").start();
command = CLIENT.commands().waitFor(command);

if ( command.isSuccess() ) {
  LOGGER.info("HDFS cluster started successfully!");
} else {
  LOGGER.info("Error starting cluster!");
}    

{% endhighlight %}

**Other services**

You can start other supported service in a similar way: 

* create the service
* assign service roles to machines
* configure the service
* start it

## Conclusions

Cloudera Manager is a nice product and simplifies deploying Hadoop a lot. They also added an API which enables developers to automate tasks. 
The main driver for this API is to monitor and check cluster status (only available in CM Enterprise) but you can do more with it. 
We use it in Axemblr to deliver Hadoop on Cloud infrastructure because it simplifies configuring and deploying Hadoop. 

Feel free to submit pull-requests to the client.

##Resources

* [Github sources](https://github.com/axemblr/cloudera-manager-api)
* [Official API documentation](http://cloudera.github.com/cm_api/apidocs/v1/index.html)
* [API tutorial](http://cloudera.github.com/cm_api/apidocs/v1/tutorial.html)
* [Python API client](http://cloudera.github.com/cm_api/)
* [Python API tutorial](http://www.cloudera.com/blog/2012/09/automating-your-cluster-with-cloudera-manager-api/)
* [Vagrant](http://vagrantup.com/)
* [VirtualBox](https://www.virtualbox.org/)
* [Cloudera Manager](http://www.cloudera.com/products-services/tools/)
