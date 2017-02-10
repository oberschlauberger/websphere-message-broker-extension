ibm-websphere-msg-broker-monitor
================================

Use Case
--------

The IBM Integration Bus, formerly known as the IBM WebSphere Message Broker Family, provides a variety of options for implementing a 
universal integration foundation based on an enterprise service bus (ESB). Implementations help to enable connectivity and transformation 
in heterogeneous IT environments for businesses of any size, in any industry and covering a range of platforms including cloud and z/OS.

This extension extracts resource and message flow statistics. The list of metrics that it extracts can be found here:

https://www-01.ibm.com/support/knowledgecenter/SSMKHH_10.0.0/com.ibm.etools.mft.doc/bn43250_.htm
https://www.ibm.com/support/knowledgecenter/en/SSMKHH_10.0.0/com.ibm.etools.mft.doc/ac19020_.htm

Prerequisites
--------------

This extension requires a AppDynamics Java Machine Agent installed and running. 

If this extension is configured for *Client* transport type (more on that later), please make sure the MQ's host and port is accessible. 
 
If this extension is configured for *Bindings* transport type (more on that later), admin level credentials to the queue manager would be needed. If the hosting OS for IBM MQ is Windows, Windows user credentials will be needed. 

Dependencies
------------

**This extension requires the IBM MQ JMS client classes.** Refer to the IBM documentation for the specific jars.
For IBM MQ 8.0.0.5, copy the following jars to the *<Machine_Agent_Dir>/WMBMonitor/lib* directory:

```
com.ibm.mq.allclient.jar
com.ibm.mq.traceControl.jar
fscontext.jar
jms.jar
providerutil.jar
```

Rebuilding the Project
----------------------

1. Clone the repo ibm-websphere-msg-broker-monitor from GitHub: https://github.com/Appdynamics
2. Create a *lib* folder in the cloned repository and insert all the required jar files from your IBM MQ installation directory.
   For IBM MQ 8.0.0.5, the required jars can be found in *<IBM_MQ_INSTALL_DIR>/java/lib*.
3. Run `mvn clean install -Pmq8` for MQ version 8.X (default) or `mvn clean install -Pmq7.5` for MQ version 7.5.
4. The *WMBMonitor-<version>.zip* should get built and found in the *target* directory.


Installation
-------------

To install the extension, extract the zip file to *<Machine_Agent_Dir>/monitors/* and then copy the required jar files to *<Machine_Agent_Dir>/monitors/WMBMonitor/lib*.
Create a *lib* folder if not already present.

There are two configurations needed:

 1. On the WebSphere Message Broker
     
    To get resource statistics from the broker, first you will have to enable the resource statistics on WMB (WebSphere Message Broker).
    There are two ways that you can enable statistics:

    a. You can run "mqsichangeresourcestats" by running it in IBM Integration Console (in WMB 9.0) or  IBM WMB Command Console (in previous versions). 
    A sample command can be 
        
    ```
    mqsichangeresourcestats BrokerA -c active -e default 
             
    where
    BrokerA -> the broker name
    default -> the execution group / integration server
    ```

    Please follow the below documentation to get more familiar with the mqsichangeresourcestats and mqsichangeflowstats command.  
    https://www-01.ibm.com/support/knowledgecenter/SSMKHH_10.0.0/com.ibm.etools.mft.doc/bj43320_.htm  
    https://www.ibm.com/support/knowledgecenter/SSMKHH_10.0.0/com.ibm.etools.mft.doc/an28420_.htm
    
    b. You can also enable resource statistics from IBM WebSphere MQ Explorer.  
      1. Open IBM WebSphere MQ Explorer
      2. Click on Integration Node i.e. Broker Name.
      3. Right click on the execution group which you want statistics for and start resource statistics. 

    To get message flow statistics, you have to enable them for a particular application / message flow:

    ```
    mqsichangeflowstats BrokerA -e default -k application -f messageflow -s -o xml -c active

    where
    BrokerA -> the broker name
    default -> the execution group / integration server
    application -> the application name
    messageflow -> the name of the message flow
    ```
      
    Once you have started the statistic collection, you can confirm it by viewing the statistics as follows 
     
        - Open IBM WebSphere MQ Explorer
        - Click on Integration Node i.e. Broker Name.
        - Right click on the execution group which you want statistics for and view resource statistics.

    Alternatively you can verify that the collection of statistics is enabled using the Broker's Web Interface.   
      
    The resource metrics get published every 20 seconds to a topic. For the above command, the statistics will get published on this topic *$SYS/Broker/BrokerA/ResourceStatistics/default*.  
    Similarly, snapshot statistics for message flows get published every 20 seconds to the corresponding topic *$SYS/Broker/+/StatisticsAccounting/SnapShot/application/messageflow*.
          
    For more details, please follow the IBM documentation:  
    http://www-01.ibm.com/support/knowledgecenter/SSMKHH_10.0.0/com.ibm.etools.mft.doc/aq20080_.htm
    
    The extension subscribes to a particular topic in a queue manager's queue. Please confirm that you have all of the following in the IBM WebSphere MQ explorer:
        
        a. A Queue Manager.
        b. A TCP Listener on some port controlled/managed by Queue Manager.  
           This port will be the port which we will use to subscribe to the statistics from the extension.
    
    If the queue manager and/or TCP listener is not present, please create them.
      
 2. On the appdynamics extension
 
    Configure the config.yml file in the WMBMonitor directory.
 
    ```
    #This will create the metrics in all the tiers,under this path
    #metricPrefix: "Custom Metrics|WMB"

    #This will create the metrics in a specific Tier/Component. Make sure to replace the
    #<COMPONENT_ID> with the appropriate one from your environment. To find the <COMPONENT_ID> in your environment
    #,please follow the screenshot here https://docs.appdynamics.com/display/PRO42/Build+a+Monitoring+Extension+Using+Java
    metricPrefix: "Server|Component:<COMPONENT_ID>|Custom Metrics|WMB"

    queueManagers:
    - host: "localhost"
        #TCP Listener port. Please make sure that the listener is started.
        port: 2414
        #Actual name of the queue manager.
        name: "QMgr1"
        #Channel name of the queue manager. Channel should be server-conn type. Make sure the channel is active.
        channelName: "SYSTEM.ADMIN.SVRCONN"
        #The transport type for the queue manager connection, the default is "Bindings".
        #For bindings type connection WMQ extension (i.e machine agent) need to be on the same machine on which WebsphereMQ server is running
        #for client type connection WMQ extension is remote to the WebsphereMQ server. For client type, Change it to "Client".
        transportType: "Bindings"
        #userId and password with admin level credentials. For Windows, please provide the administrator credentials.
        userID: ""
        password: ""
        #SSL related properties. Please provide the path.
        sslKeyRepository: ""
        #Cipher Suites. eg. "SSL_RSA_WITH_AES_128_CBC_SHA256"
        cipherSuite: ""
        clientID: "wmb_appd_ext"

        #This extension extracts resource statistics as listed
        #http://www.ibm.com/support/knowledgecenter/SSMKHH_10.0.0/com.ibm.etools.mft.doc/bn43250_.htm
        #Please make sure to provide unique subscriber name for each statistic subscription.
        metrics:
        resourceStatistics:
            - name: "$SYS/Broker/+/ResourceStatistics/#"
            subscriberName: "allResourceStatistics"
        flowStatistics:
            - name: "$SYS/Broker/+/StatisticsAccounting/SnapShot/#/#"
            subscriberName: "allFlowStatistics"

        #For flow metrics, all the fields listed in this section will be reported as metrics.
        #Available fields:
        #https://www.ibm.com/support/knowledgecenter/en/SSMKHH_10.0.0/com.ibm.etools.mft.doc/ac19020_.htm
        flowMetricFields:
        messageFlowFields:
            - TotalElapsedTime
            - MaximumElapsedTime
            - MinimumElapsedTime
            - TotalCPUTime
            - MaximumCPUTime
            - MinimumCPUTime
            - CPUTimeWaitingForInputMessage
            - ElapsedTimeWaitingForInputMessage
            - TotalInputMessages
            - TotalSizeOfInputMessages
            - MaximumSizeOfInputMessages
            - MinimumSizeOfInputMessages
            - NumberOfThreadsInPool
            - TimesMaximumNumberofThreadsReached
            - TotalNumberOfMQErrors
            - TotalNumberOfMessagesWithErrors
            - TotalNumberOfErrorsProcessingMessages
            - TotalNumberOfTimeOutsWaitingForRepliesToAggregateMessages
            - TotalNumberOfCommits
            - TotalNumberOfBackouts
        threadFields: 
            - Number
            - TotalNumberOfInputMessages
            - TotalElapsedTime
            - TotalCPUTime
            - CPUTimeWaitingForInputMessage
            - ElapsedTimeWaitingForInputMessage
            - TotalSizeOfInputMessages
            - MaximumSizeOfInputMessages
            - MinimumSizeOfInputMessages
        nodeFields:
            - TotalElapsedTime
            - MaximumElapsedTime
            - MinimumElapsedTime
            - TotalCPUTime
            - MaximumCPUTime
            - MinimumCPUTime
            - CountOfInvocations
            - NumberOfInputTerminals
            - NumberOfOutputTerminals
        terminalFields:
            - CountOfInvocations
        
        #Some derived fields are available for flow metrics.
        derivedFlowMetricFields:
        messageFlowFields:
            - AverageElapsedTime
            - AverageCPUTime
            - AverageCPUTimeWaitingForInputMessage
            - AverageElapsedTimeWaitingForInputMessage
            - AverageSizeOfInputMessages
        threadFields:
            - AverageElapsedTime
            - AverageCPUTime
            - AverageCPUTimeWaitingForInputMessage
            - AverageElapsedTimeWaitingForInputMessage
            - AverageSizeOfInputMessages
        nodeFields:
            - AverageElapsedTime
            - AverageCPUTime
            - AverageSizeOfInputMessages
            
    numberOfThreads: 10        
    ```
    
    In the above config, *port* is the port on which the TCP listener was started in the IBM WebSphere MQ Explorer. By default, the extension will get statistics for all the execution groups, applications, and message flows on a particular broker. The "#" wildcard in the topic name signifies that. If you want to view statistics for selected execution groups, you can configure as below:

    ```
    # Topics for particular execution groups belonging to a broker.
    resourceStatTopics:
        - name: "$SYS/Broker/BrokerA/ResourceStatistics/execGroupA"
          subscriberName: "execGroupA"
        - name: "$SYS/Broker/BrokerA/ResourceStatistics/execGroupB"
          subscriberName: "execGroupB"
    flowStatistics:
        - name: "$SYS/Broker/+/StatisticsAccounting/SnapShot/applicationA/messageFlowA"
          subscriberName: "flowAStatistics"
    ```
  
Troubleshooting
---------------

1. Verify Machine Agent Data: Please start the Machine Agent without the extension and make sure that it reports data. Verify that the machine agent status is UP and it is reporting Hardware Metrics.
2. config.yml: Validate the file [here](http://www.yamllint.com/)
3. MQ Version incompatibilities :  In case of any jar incompatibility issue, the rule of thumb is to use the jars from MQ version 8.0.  
4. Metric Limit: Please start the machine agent with the argument `-Dappdynamics.agent.maxMetrics=5000` if there is a metric limit reached error in the logs. If you don't see the expected metrics, this could be the cause.
5. Check Logs: There could be some obvious errors in the machine agent logs. Please take a look.
6. `The config cannot be null` error.
   This usually happenes when on a windows machine in monitor.xml you give config.yaml file path with linux file path separator */*. Use Windows file path separator *\* e.g. *monitors\MQMonitor\config.yaml*. For Windows, please specify 
   the complete path

7. Collect Debug Logs: Edit the file, *<MachineAgent>/conf/logging/log4j.xml* and update the level of the appender *com.appdynamics* and *com.singularity* to debug. Let it run for 5-10 minutes and attach the logs to a support ticket

WorkBench
---------

Workbench is a feature by which you can preview the metrics before registering it with the controller. This is useful if you want to fine tune the configurations. Workbench is embedded into the extension jar.
To use the workbench

1. Follow all the installation steps
2. Start the workbench with the command

```
    java -jar /path/to/MachineAgent/monitors/WMBMonitor/ibm-websphere-msg--broker-extension.jar
```

  This starts an http server at http://host:9090/. This can be accessed from the browser.
3. If the server is not accessible from outside/browser, you can use the following end points to see the list of registered metrics and errors.

```
    #Get the stats
    curl http://localhost:9090/api/stats
    #Get the registered metrics
    curl http://localhost:9090/api/metric-paths
```
4. You can make the changes to config.yml and validate it from the browser or the API
5. Once the configuration is complete, you can kill the workbench and start the Machine Agent


## Custom Dashboard ##
![](https://raw.githubusercontent.com/Appdynamics/ibm-websphere-msg-broker-monitor/master/ibm-wmb.png)

## Contributing ##

Always feel free to fork and contribute any changes directly via [GitHub][].

## Community ##

Find out more in the [AppDynamics Exchange][].

## Support ##

For any questions or feature request, please contact [AppDynamics Center of Excellence][].

**Version:** 3.0.0
**Controller Compatibility:** 4.0+
**IBM WebSphere Message Broker Version Tested On:** 8.0.0.0

[Github]: https://github.com/Appdynamics/ibm-websphere-msg-broker-monitor
[AppDynamics Exchange]: http://community.appdynamics.com/t5/AppDynamics-eXchange/idb-p/extensions
[AppDynamics Center of Excellence]: mailto:help@appdynamics.com