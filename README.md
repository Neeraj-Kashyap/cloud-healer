[![Codacy Badge](https://api.codacy.com/project/badge/Grade/3ce697b78d8742b89613dc02ef9021d5)](https://www.codacy.com/app/romanovf/cloud-healer?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=tomtom-international/cloud-healers&amp;utm_campaign=Badge_Grade)
[![Build Status](https://img.shields.io/travis/tomtom-international/cloud-healer.svg?maxAge=3600&branch=master)](https://travis-ci.org/tomtom-international/cloud-healer)
[![License](http://img.shields.io/badge/license-APACHE2-blue.svg)]()
[![Release](https://img.shields.io/github/release/tomtom-international/cloud-healer.svg?maxAge=3600)](https://github.com/tomtom-international/cloud-healer/releases)
 
**Copyright (C) 2012-2017, TomTom International BV. All rights reserved.**

## Introduction
TomTom VM Active Self Recycler enables active cloud-agnostic self-managed instance recycling for Java. This component was developed to 
 address following unique operational requirements:
  * Creating computing instance takes substantial amount of time, e.g. several GBs needs to be copied. 
  * Your VM(computing instance) can enter unstable state, e.g. connectivity to external resources(process, DB, message queues, e.t.c.) is lost.
The error state can happen unpredictably and can not be scheduled. If you need, for example, self-terminate running instance after certain period of time,
 you can more easily achieve this by leveraging cloud provider facilities, e.g. AWS provides that ability by tuning _cloud-init_ script.
  * Restarting instance most likely solves the issue. It can also include restarting external process if they are co-located. 
  * Load-Balancer(LB) does not support automatic VM recovery(Azure case).
  
## TomTom VM Active Self Recycler features
 * Enables graceful instance recycling without impacting users.
 * Supports Azure and AWS.
 * Shields client code from cloud-specific SDK dependencies and low-level details of cloud infrastructure.
 * Can send notifications to topic(AWS) and EventHub(Azure).
 * Thread management, forks new thread for recycling.
 
## Why standard LB features are not enough?
Firstly, unhealthy instance gets removed from the cluster which might cause performance degradation as spinning off fresh instance is slow(see requirements).
 Relying on LB to detect and recycle instance is _passive_ approach. LB expect predefined number of consecutive health
check failures for VM to become unhealthy. E.g. By default, Azure LB executes health check every 15 sec, 3 consecutive health
check must be failed for instance to remove it from LB. 
Following pictures depict how LB handles node failure

                               +---------------+
     CLIENTS ->                | Load Balancer |
                               +---------------+
                                     /  \               
                      +-------------+    +------------------+
     AUTOSCALE        | "Node1"     |    | "Node2"          |
     GROUP            | Status:OK   |    |  Status:OK       |
                      +-------------+    +------------------+
                      /                               \
              +------------+                      +------------------+      
     EXTERNAL | "Node 1"   |                      | "Node 2"         |      
     RESOURCE | Dependency |                      | Dependency       |
              +------------+                      +------------------+

Now _Node 2_ loses connectivity to external resource and enters unstable state and becomes unhealthy:

                               +---------------+
     CLIENTS ->                | Load Balancer |
                               +---------------+
                                     /  \               
                      +-------------+    +------------------+
     AUTOSCALE        | "Node 1"    |    | "Node 2"         |
     GROUP            | Status:OK   |    |  Status:ERROR    |
                      +-------------+    +------------------+
                      /                               
              +------------+                      +------------------+      
     EXTERNAL | "Node 1    |                      | "Node 2"         |      
     RESOURCE | Dependency |                      | Dependency       |
              +------------+                      +------------------+
After some time LB detects _Node 2_ failure, removes it from cluster and, if supported by cloud provider, spins off new one.
Notice that during that period only _Node 1_ serves all requests which could cause 'snowball' effect on overall system performance:

                                  +---------------+
    CLIENTS(can see degradation)->| Load Balancer |
                                  +---------------+
                                     /                 
                      +-------------+    +------------------+
     AUTOSCALE        | "Node 1"    |    | "Node 3"         |
     GROUP            | Status:OK   |    |  Status:STARTING |
                      +-------------+    +------------------+
                      /                               
              +------------+                      +------------------+      
     EXTERNAL | "Node 1"   |                      | "Node 3"         |      
     RESOURCE | Dependency |                      | Dependency       |
              +------------+                      +------------------+      
When _Node 3_ is ready(could take up to several minutes) LB adds it to the cluster and starts routing traffic to it:

                               +---------------+
     CLIENTS(back to normal)-> | Load Balancer |
                               +---------------+
                                     /  \               
                      +-------------+    +------------------+
     AUTOSCALE        | "Node 1"    |    | "Node 3"         |
     GROUP            | Status:OK   |    |  Status:OK       |
                      +-------------+    +------------------+
                      /                               \
              +------------+                      +------------------+      
     EXTERNAL | "Node 1"   |                      | "Node 3"         |      
     RESOURCE | Dependency |                      | Dependency       |
              +------------+                      +------------------+
## How VM Active Self Recycler addresses this case
Instead of taking _passive_ approach, VM Active Self-Recycler empowers node to function _proctively_, i.e. 
the moment error condition occurs spin off new nodes(double number of instances) and terminate itself after new nodes are up and running:
_Node 2_ loses connectivity to external resource and enters error internal state. VM Active Self-Recycler starts new instance :

                               +---------------+
     CLIENTS(no degradation)-> | Load Balancer |
                               +---------------+
                                     /  \               
                      +-------------+    +---------------------+               +------------------+  
     AUTOSCALE        | "Node 1"    |    | "Node 2"            |               | "Node 3:         |
     GROUP            | Status:OK   |    |  Status:OK(UNSTABLE)|---create- ->  | Status:STARTING  |
                      +-------------+    +---------------------+               +------------------+
                      /                                                              \
              +------------+                      +------------------+         +------------------+ 
     EXTERNAL | "Node 1    |                      | "Node 2"         |         | "Node 3"         |
     RESOURCE | Dependency |                      | Dependency       |         | Dependency       |
              +------------+                      +------------------+         +------------------+
When _Node 3_ is up and running VM Active Self-Recycler replaces it in LB and triggers _Node 2_ self-termination: 

                               +---------------+
     CLIENTS ->                | Load Balancer |
                               +---------------+
                                     /  \               
                      +-------------+    +------------------+               +-------------------+  <---------|
     AUTOSCALE        | "Node 1"    |    | "Node 3"         |               | "Node 2:          |            |
     GROUP            | Status:OK   |    |  Status:OK       |               | Status:TERMINATING|->terminate-|
                      +-------------+    +------------------+               +-------------------+
                      /                              \                                
              +------------+                      +------------------+         +------------------+ 
     EXTERNAL | "Node 1    |                      | "Node 3"         |         | "Node 2"         |
     RESOURCE | Dependency |                      | Dependency       |         | Dependency       |
              +------------+                      +------------------+         +------------------+
## Build Environment (Java 8)
The source uses Java JDK 1.8, so make sure your Java compiler is set to 1.8, for example
using something like (MacOSX):

    export JAVA_HOME=`/usr/libexec/java_home -v 1.8`

## Build
To build the VM self-recycler, simply go to the root folder and then type:

    mvn clean install

or, to view the test coverage, execute:

    mvn clean verify jacoco:report
    open target/site/jacoco/index.html
## How to use TT VM Self-Recycler
 * Obtain the code _TT VM Active Self-Recycler_ code by checking git repo or downloading release version
 * build it(see section above) and 
 * pick up required target cloud provider(_AWS_ and _Azure_ are supported) and add only 2 corresponding _-recycling_ and _-config_ modules into your project dependencies, e.g.:
 * For AWS add:
 
 
         <dependency>
             <groupId>com.tomtom.cloud</groupId>
             <artifactId>aws-recycling</artifactId>
             <version>1.0.0</version>
         </dependency>
         <dependency>
             <groupId>com.tomtom.cloud</groupId>
             <artifactId>aws-config</artifactId>
             <version>1.0.0</version>
         </dependency> 
 * For Azure add:
 
 
         <dependency>
             <groupId>com.tomtom.cloud</groupId>
             <artifactId>azure-recycling</artifactId>
             <version>1.0.0</version>
         </dependency>
         <dependency>
             <groupId>com.tomtom.cloud</groupId>
             <artifactId>azure-config</artifactId>
             <version>1.0.0</version>
         </dependency>    
  * when configuring Spring web app, add two _Spring_ configurations. E.g. for Spring-boot 
 * For AWS add:
 
 
         @SpringBootApplication
         @ImportAutoConfiguration({RecyclingAutoConfig.class, AwsRecyclingAutoConfig.class})

 * For Azure add:
 
 
         @SpringBootApplication
         @ImportAutoConfiguration({RecyclingAutoConfig.class, AzureRecyclingAutoConfig.class})
  * when running your Java app, add _active.recycling.CLOUD-PROVIDER.enabled=true_ system property and other required props:
  * For AWS add:
 
 
         -Dactive_recycling_aws_enabled=true -Dactive_recycling_aws_topic=${SHUTDOWN_TOPIC}
  * For Azure add:
 
 
         -Dactive_recycling_azure_enabled=true -Dactive_recycling_azure_gateway"=${AZURE_GATEWAY} -Dactive_recycling_azure_instance_id=${OWN_INSTANCE_ID}
  * Inject _ActiveVMRecycler_ bean into your service and call _boolean ActiveVMRecycler::scaleOutAndRecycle(String reason)_ method when instance becomes unstable
  * _boolean ActiveVMRecycler::scaleOutAndRecycle(String reason)_ returns _true_ if it successfully triggered new thread and _false_ if recycling is already in progress.
## Organization of Source Code

    cloud-healer
    |
    +-- recycling-config-common
    |   |
    |   +-- RecyclingAutoConfig common props(enabling and check timeout) for active cloud instance self-recycling
    |
    +-- recycling-common
    |  |
    |  +-- WorkerRecycler          Facade for interacting with node self recycler.
    |  +-- WorkerRecyclerThread    Thread for triggering the recycling of the current instance
    |  +-- CloudAdapter            Interface to be implemented for direct interactions with cloud provider (E.g: AWS or azure).
    |                           
    +-- azure-recycling            Azure-specific recycling implementation
    |  +-- AzureCloudAdapter       
    |  
    +-- azure-config              Azure-specific recycling configuration (gateway, eventhub,e.t.c,)
    |  +-- AzureMonitoringAutoConfig       
    |
    +-- aws-recycling              AWS-specific recycling implementation
    |  
    +-- aws-config                 AWS-specific recycling configuration (topic, instance, e.t.c)
    |  +-- AwsRecyclingAutoConfig       
# License

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
