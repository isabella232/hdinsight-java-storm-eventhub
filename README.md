---
services: hdinsight
platforms: java
author: blackmist
---

# hdinsight-java-storm-eventhub

An example of how to read and write from Azure Event Hub using an Apache Storm topology (written in Java,) on an Azure HDInsight cluster. This example uses the Event Hubs spout provided by Microsoft, which is required for Storm 0.9.3 (HDInsight version 3.2.)

##Prerequisites

* An HDInsight cluster - either [Linux-based](hdinsight-apache-storm-tutorial-get-started-linux.md) or [Windows-based](hdinsight-apache-storm-tutorial-get-started.md)

* [Azure EventHubs](../eventhubs/event-hubs-csharp-ephcs-getstarted.md)

* [Oracle Java Developer Kit (JDK) version 7](https://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html) or an equivalent such as [OpenJDK](http://openjdk.java.net/)

* [Maven](https://maven.apache.org/download.cgi)

* A text editor or Java Integrated Development Environment (IDE)

##How it works

__com.microsoft.example.EventHubWriter__ writes random data to an Azure Event Hub. The data is generated by a spout, and is a random device ID and device value. So it's simulating some hardware that emits a string ID and a numeric value.

__com.microsoft.example.EventHubReader__ reads data from Event Hub (the data written by EventHubWriter,) and stores it to HDFS (WASB in this case, since this was written and tested with Azure HDInsight) in the /devicedata directory.

The data format in Event Hub is a JSON document with the following format:

    { "deviceId": "unique identifier", "deviceValue": some value }

The reason it's stored in JSON is compatibility - I recently ran into someone who wasn't formatting data sent to Event Hub as JSON (from a Java application,) and was reading it into a Java app. Worked fine. Then they wanted to replace the reading component with a C# application that expected JSON. Problem! Always store to a nice format that is future proofed in case your components change.

##Required information

* An Azure Event Hub with two shared access policies; one that has __listen__ permissions, and one that has __write__ permissions. I will refer to these as "reader" and "writer", which is what I named mine

    * The policy keys for the "reader" and "writer" policies

    * The name of your Event Hub

    * The Service Bus namespace that your Event Hub was created in

    * The number of partitions available with your Event Hub configuration
    
    For information on creating and using EventHubs, see the Create an Event Hub section of [Get Started with EventHubs](https://azure.microsoft.com/documentation/articles/event-hubs-java-storm-getstarted/#create-an-event-hub)

* The Azure Storage account that is the default storage for your HDInsight cluster

    * The access key for the storage account

    * The container name that is the default storage for your HDInsight cluster

    You can find the storage account and access key information by going to the Azure Portal and selecting your HDInsight cluster. From the cluster properties blade, select __All settings__, and then use __Azure storage keys__ to drill down to the storage account and key information.
    
    The container name is usually the same as the name of the cluster. If you used a different default container name when creating the cluster, use the value you specified.

##Confgure and build

1. Fork & clone the repository so you have a local copy.

2. Use the following to install a couple components from the /lib directory of this repo into your local Maven repository. These are required for communicating with Azure Event Hubs and the WASB storage used by HDInsight. Eventually these will be included in the Hadoop/Storm bits on Maven

		mvn -q install:install-file -Dfile=lib/eventhubs/eventhubs-storm-spout-0.9.4-jar-with-dependencies.jar -DgroupId=com.microsoft.eventhubs -DartifactId=eventhubs-storm-spout -Dversion=0.9.4 -Dpackaging=jar

		mvn -q org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=lib/hadoop/hadoop-azure-3.0.0-SNAPSHOT.jar

		mvn -q org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=lib/hadoop/hadoop-client-3.0.0-SNAPSHOT.jar

		mvn -q org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=lib/hadoop/hadoop-hdfs-3.0.0-SNAPSHOT.jar

		mvn -q org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=lib/hadoop/hadoop-common-3.0.0-SNAPSHOT.jar -DpomFile=lib/hadoop/hadoop-common-3.0.0-SNAPSHOT.pom

		mvn -q org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=lib/hadoop/hadoop-project-dist-3.0.0-SNAPSHOT.pom -DpomFile=lib/hadoop/hadoop-project-dist-3.0.0-SNAPSHOT.pom

		mvn -q org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=lib/hadoop/hadoop-project-3.0.0-SNAPSHOT.pom -DpomFile=lib/hadoop/hadoop-project-3.0.0-SNAPSHOT.pom

		mvn -q org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=lib/hadoop/hadoop-main-3.0.0-SNAPSHOT.pom -DpomFile=lib/hadoop/hadoop-main-3.0.0-SNAPSHOT.pom

	NOTE: If you're using Powershell, you mau have to put the -Dfoo=bar parmeters in quotes. The whole thing; "-Dfoo=bar".

	Also note that I got these files from https://github.com/hdinsight/hdinsight-storm-examples, so you should look there for the latest versions.

2. Add the Event Hub configuration to the /conf/EventHubs.properties file. This is used to configure the spout that reads from Event Hub and the bolt that writes to it.

3. Add the storage account information to the /conf/core-site.xml file. This is used to tell the HDFS-bolt how to talk to HDInsight WASB, which is backed by Azure Storage.

4. Use `mvn package` to build everything.

    Once the build completes, the `target` directory will contain a file named `EventHubExample-1.0-SNAPSHOT.jar`.

##Deploy

The jar created by this project contains two topologies; __com.microsoft.example.EventHubWriter__ and __com.microsoft.example.EventHubReader__. The EventHubWriter topology should be started first, as it writes events in to Event Hub that are then read by the EventHubReader.

###If using a Linux-based cluster

1. Use SCP to copy the jar package to your HDInsight cluster. Replace USERNAME with the SSH user for your cluster. Replace CLUSTERNAME with the name of your HDInsight cluster:

        scp ./target/EventHubExample-1.0-SNAPSHOT.jar USERNAME@CLUSTERNAME-ssh.azurehdinsight.net:.

    If you used a password for your SSH account, you will be prompted to enter the password. If you used an SSH key with the account, you may need to use the `-i` parameter to specify the path to the key file. For example, `scp -i ~/.ssh/id_rsa ./target/EventHubExample-1.0-SNAPSHOT.jar USERNAME@CLUSTERNAME-ssh.azurehdinsight.net:.`.

    If your client is a Windows workstation, you may not have an SCP command installed. I recommend PSCP, which can be downloaded from the [PuTTY download page](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html).

    This command will copy the file to the home directory of your SSH user on the cluster.

1. Once the file has finished uploading, use SSH to connect to the HDInsight cluster. Replace **USERNAME** the the name of your SSH login. Replace **CLUSTERNAME** with your HDInsight cluster name:

        ssh USERNAME@CLUSTERNAME-ssh.azurehdinsight.net

    If you used a password for your SSH account, you will be prompted to enter the password. If you used an SSH key with the account, you may need to use the `-i` parameter to specify the path to the key file. The following example will load the private key from `~/.ssh/id_rsa`:
    
    `ssh -i ~/.ssh/id_rsa USERNAME@CLUSTERNAME-ssh.azurehdinsight.net`

    If you are using PuTTY, enter `CLUSTERNAME-ssh.azurehdinsight.net` in the __Host Name (or IP address)__ field, and then click __Open__ to connect. You will be prompted to enter your SSH account name.

    If you used a password for your SSH account, you will be prompted to enter the password. If you used an SSH key with the account, you may need to use the following steps to select the key:
    
    1. In **Category**, expand **Connection**, expand **SSH**, and select **Auth**.
    2. Click **Browse** and select the .ppk file that contains your private key.
    3. Click __Open__ to connect.

2. Use the following command to start the topologies:

        storm jar EventHubExample-1.0-SNAPSHOT.jar com.microsoft.example.EventHubWriter writer
        storm jar EventHubExample-1.0-SNAPSHOT.jar com.microsoft.example.EventHubReader reader

    This will start the topologies and give them a friendly name of "reader" and "writer".

3. Wait a minute or two to allow the topologies to write and read events from event hub, then use the following command to verify that the EventHubReader is storing data to your HDInsight storage:

        hadoop fs -ls /devicedata

    This should return a list of files similar to the following:

        -rw-r--r--   1 storm supergroup      10283 2015-08-11 19:35 /devicedata/wasbbolt-14-0-1439321744110.txt
        -rw-r--r--   1 storm supergroup      10277 2015-08-11 19:35 /devicedata/wasbbolt-14-1-1439321748237.txt
        -rw-r--r--   1 storm supergroup      10280 2015-08-11 19:36 /devicedata/wasbbolt-14-10-1439321760398.txt
        -rw-r--r--   1 storm supergroup      10267 2015-08-11 19:36 /devicedata/wasbbolt-14-11-1439321761090.txt
        -rw-r--r--   1 storm supergroup      10259 2015-08-11 19:36 /devicedata/wasbbolt-14-12-1439321762679.txt

    Some files may show a size of 0, as they have been created by the EventHubReader, but data has not been stored to them yet.

    You can view the contents of these files by using the following command:

        hadoop fs -text /devicedata/*.txt

    This will return data similar to the following:

        3409e622-c85d-4d64-8622-af45e30bf774,848981614
        c3305f7e-6948-4cce-89b0-d9fbc2330c36,-1638780537
        788b9796-e2ab-49c4-91e3-bc5b6af1f07e,-1662107246
        6403df8a-6495-402f-bca0-3244be67f225,275738503
        d7c7f96c-581a-45b1-b66c-e32de6d47fce,543829859
        9a692795-e6aa-4946-98c1-2de381b37593,1857409996
        3c8d199b-0003-4a79-8d03-24e13bde7086,-1271260574

    The first column contains the device ID value and the second column is the device value.

4. Use the following commands to stop the topologies:

        storm kill reader
        storm kill writer

###If using a Windows-based cluster

1. Open your browser to https://CLUSTERNAME.azurehdinsight.net. When prompted, enter the administrator credentials for your HDInsight cluster. You will arrive at the Storm Dashboard.

2. Use the __Jar File__ dropdown to browse and select the EventHubExample-1.0-SNAPSHOT.jar file from your build environment.

3. For __Class Name__, enter `com.mirosoft.example.EventHubWriter`.

4. For __Additional Parameters__, enter `writer`. Finally, click __Submit__ to upload the jar and start the EventHubWriter topology.

5. Once the topology has started, use the form to start the EventHubReader:

    * __Jar File__: select the EventHubExample-1.0-SNAPSHOT.jar that was previously uploaded
    * __Class Name__: enter `com.microsoft.example.EventHubReader`
    * __Additional Parameters__: enter `reader`

    Click submit to start the EventHubReader topology.

6. Wait a few minutes to allow the topologies to generate events and store then to Azure Storage, then select the __Query Console__ tab at the top of the __Storm Dashboard__ page.

7. On the __Query Console__, select __Hive Editor__ and replace the default `select * from hivesampletable` with the following:

        create external table devicedata (deviceid string, devicevalue int) row format delimited fields terminated by ',' stored as textfile location 'wasb:///devicedata/';
        select * from devicedata limit 10;

    Click __Select__ to run the query. This will return 10 rows from the data written to Azure Storage (WASB) by the EventHubReader. Once the query completes, you should see data similar to the following:

        3409e622-c85d-4d64-8622-af45e30bf774,848981614
        c3305f7e-6948-4cce-89b0-d9fbc2330c36,-1638780537
        788b9796-e2ab-49c4-91e3-bc5b6af1f07e,-1662107246
        6403df8a-6495-402f-bca0-3244be67f225,275738503
        d7c7f96c-581a-45b1-b66c-e32de6d47fce,543829859
        9a692795-e6aa-4946-98c1-2de381b37593,1857409996
        3c8d199b-0003-4a79-8d03-24e13bde7086,-1271260574

8. Select the __Storm Dashboard__ at the top of the page, then select __Storm UI__. From the __Storm UI__, select the link for the __reader__ topology and then use the __Kill__ button to stop the topology. Repeat the process for the __writer__ topology.

### Checkpointing

The EventHubSpout periodically checkpoints its state to the Zookeeper node, which saves the current offset for messages read from the queue. This allows the component to start receiving messages at the saved offset in the following scenarios:

* The component instance fails and is restarted.

* You grow or shrink the cluster by adding or removing nodes.

* The topology is killed and restarted **with the same name**.

####On Windows-based HDInsight clusters

You can export and import the persisted checkpoints to WASB (the Azure Storage used by your HDInsight cluster.) The scripts to do this are located on the Storm on HDInsight cluster, at **c:\apps\dist\storm-0.9.3.2.2.1.0-2340\zkdatatool-1.0\bin**.

The version number in the path may be different, as the version of Storm installed on the cluster may change in the future.

The scripts in this directory are:

* **stormmeta_import.cmd**: Import all Storm metadata from the cluster default storage container into Zookeeper.

* **stormmeta_export.cmd**: Export all Storm metadata from Zookeeper to the cluster default storage container.

* **stormmeta_delete.cmd**: Delete all Storm metadata from Zookeeper.

Export an import allows you to persist checkpoint data when you need to delete the cluster, but want to resume processing from the current offset in the hub when you bring a new cluster back online.

Since the data is persisted to the default storage container, the new cluster **must** use the same storage account and container as the previous cluster.

##How real world is this?

Since it's an example, there are some things that you might want to tweak. Noticably it has no error checking for someone putting bad data in Event Hub. It also has a size of 20kb for the files written to WASB storage. So I would recommend adding error checking and figuring out what the ideal file write size is for your scenario.
