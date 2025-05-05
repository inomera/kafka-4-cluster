# kafka-4-cluster  

* 🚀 **Apache Kafka** is running in **KRaft mode by default** – Bye Bye Zookeeper!
* 🔧 **Simplicity first!**
* The Kafka server (node) configuration is motivated by a balanced focus on:
    * 📈 **High throughput**
    * ⚡ **Low latency**
    * 🛡️ **Moderate durability** – all tuned to maintain overall **system stability** at a medium scale.

 * 🚀 New Release Summary News

ZooKeeper Removal: Kafka 4.0 runs fully in KRaft mode by default, eliminating the need for ZooKeeper, which simplifies operations and improves scalability.

New Consumer Group Protocol (KIP-848): This new protocol improves rebalance performance, reducing downtime and latency in large-scale deployments.

Early Access to Queues (KIP-932): Kafka now supports queue semantics, enabling point-to-point messaging use cases natively.

Java Version Requirements: Kafka Clients and Streams now require Java 11, while Brokers, Connect, and Tools require Java 17.

Deprecated APIs Removed: Kafka 4.0 removes APIs deprecated for over a year and defines new minimum supported client/broker versions (per KIP-896).
https://cwiki.apache.org/confluence/display/KAFKA/KIP-896%3A+Remove+old+client+protocol+API+versions+in+Kafka+4.0


![Kraft Quorum](docs/kafka4-cluster-with-raft.png "Kraft Quorum")

  - A major feature that we are introducing with 3.0 is the ability for KRaft Controllers and KRaft Brokers to generate, replicate, and load snapshots for the metadata topic partition named __cluster_metadata. 
  - The major feature of the 3.0.0 version is coming by default mode

# Do the same process for three linux servers.

Assume that the servers' ips (10.123.61.61,10.123.61.62,10.123.61.63)

Directory hierarchy

    /app/jdk #jdk folder

    /app/kafka4 #kafka folder

All scripts' paths are depended on the directory hierarchy.

# 1 - Download openjdk 23

JDK is a pre-requisite for running kafka

download jdk 23 from the link. (our installation is for linux operating system distribution)

https://download.java.net/openjdk/jdk23/ri/openjdk-23+37_linux-x64_bin.tar.gz

    tar -zxvf openjdk-23+37_linux-x64_bin.tar.gz

    mv jdk-23 /app/jdk

# 2 - Download kafka

download kafka 4.0.0 binary distribution from official site.

https://kafka.apache.org/downloads

https://dlcdn.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz

    tar -zxvf kafka_2.13-4.0.0.tgz

    mv kafka_2.13-4.0.0 kafka4

# 3 Create directories for logs, data logs, scripts

create directory for kafka server logs

    mkdir /app/kafka4/logs

create directory for kafka messaging, metadata logs (important logs)

    mkdir /app/kafka4/data/metadata-logs

    mkdir /app/kafka4/data/kraft-combined-logs

create directory for scripts

    mkdir /app/kafka4/scripts

# 4 Configure kafka kraft server properties

    cd /app/kafka4/config/

edit the server1.properties using vim or nano editor.

#Important Configs for the kraft cluster

The role of this server. Setting this puts us in KRaft mode. A node(server) can act as broker, controller or both. The server acts as both a broker and a controller in KRaft mode the below config.
    
    process.roles=broker,controller

The node id associated with this instance's roles. It should be unique for each node.

    node.id=1

The connection string for the controller quorum

    controller.quorum.voters=1@10.123.61.61:19092,2@10.123.61.62:19092,3@10.123.61.63:19092

The address the socket server listens on. The server ip for broker and listener.
broker use 9092 port for clients, controller use 19092 port for quorum in cluster (leader election, data replication)

    listeners=PLAINTEXT://10.123.61.61:9092,CONTROLLER://10.123.61.61:19092
    inter.broker.listener.name=PLAINTEXT

A comma separated list of directories under which to store log files

    log.dirs=/app/kafka4/data/kraft-combined-logs
    
    metadata.log.dir=/app/kafka4/data/metadata-logs
    
The default number of log partitions per topic. More partitions allow greater
parallelism for consumption, but this will also result in more files across
the brokers. The partition number should be proper of cluster node number.

    num.partitions=6

#kraft quorum properties

quorum.voters: This is a connection map which contains the IDs of the voters and their respective endpoint. We use the following format for each voter in the list {broker-id}@{broker-host):{broker-port}. For example, `quorum.voters=1@kafka-1:9092, 2@kafka-2:9092, 3@kafka-3:9092`.

quorum.fetch.timeout.ms: Maximum time without a successful fetch from the current leader before a new election is started.

quorum.election.timeout.ms: Maximum time without collected a majority of votes during the candidate state before a new election is retried.

quorum.election.backoff.max.ms: Maximum exponential backoff time (based on the number if retries) after an election timeout, before a new election is triggered.

quorum.request.timeout.ms: Maximum time before a pending request is considered failed and the connection is dropped.

quorum.retry.backoff.ms: Initial delay between request retries. This config and the one below is used for retriable request errors or lost connectivity and are different from the election.backoff configs above.

quorum.retry.backoff.max.ms: Max delay between requests. Backoff will increase exponentially beginning from quorum.retry.backoff.ms (the same as in KIP-580).

broker.id: The existing broker id config shall be used as the voter id in the Raft quorum.

Each node of cluster server1.properties is under the node-{number} folders.

# 5 Generate  the cluster id

    ./bin/kafka-storage.sh random-uuid

Cluster ID : b42Uz-P1SEyPl5jQdNakTm

# 6 Format the storage directories

    ./bin/kafka-storage.sh format -t C4H8exX8SxWNPwRirJwleQ -c ./config/server.properties

Output :

Formatting metadata directory /app/kafka4/data/metadata-logs with metadata.version 4.0-IV3.
Formatting data directory /app/kafka4/data/kraft-combined-logs with metadata.version 4.0-IV3.

Use the cluster id formatting for all nodes.

# 7 kafka bash script for operational process

start, stop, kill, log processes

The kafka.sh file is under the docs.

```bash

#sh_aliases and add following 2 lines:
#
# alias kafka="$KAFKA_SCRIPTS/kafka.sh kafka"
#
# Then use with the following commands:
#
# kafka start|stop|log
JAVA_HOME=/app/jdk/openjdk-23.0.2
export PATH=$PATH:$JAVA17_HOME/bin;

KAFKA_DIR="/app/kafka4"
LOG_DIR="${KAFKA_DIR}/logs"

KAFKA_LOG="${LOG_DIR}/kafka.out"

PROGRAM="$1"
COMMAND="$2"

kafkaStart() {
    echo "Starting kafka..."
    rm "${KAFKA_LOG}"
    nohup bash "$KAFKA_DIR/bin/kafka-server-start.sh" "$KAFKA_DIR/config/server.properties" >"${KAFKA_LOG}" 2>&1 &
    sleep 2
    echo "Probably it is started..."
        kafkaLog
}

kafkaLog() {
        less +F "${KAFKA_LOG}"
}

kafkaStop() {
        echo "Stopping kafka..."
        PIDS=$(ps -ef | grep "$KAFKA_DIR/config/server.properties" | grep kafka | grep java | grep -v grep | awk {'print $2'})
        echo "Kill kafka with process id ${PIDS}"
        kill -s TERM ${PIDS}
}

kafkaPid() {
        PIDS=$(ps -ef | grep "$KAFKA_DIR/config/server.properties" | grep kafka | grep java | grep -v grep | awk {'print $2'})
        echo "Kafka PID = ${PIDS}"
}

kafkaKill() {
        echo "Killing kafka..."
        PIDS=$(ps -ef | grep "$KAFKA_DIR/config/server.properties" | grep kafka | grep java | grep -v grep | awk {'print $2'})
        echo "Kill kafka with process id ${PIDS}"
        kill -9 ${PIDS}
}

if [ -z "$PROGRAM" ] || [ -z "$COMMAND" ] ; then
        echo "Usage kafka start|stop|log|pid|kill"
        exit 1
elif [ "$PROGRAM" != "kafka" ] ; then
        echo "Invalid program argument: ${PROGRAM}"
        exit 1
elif [ "$COMMAND" != "start" ] && [ "$COMMAND" != "log" ] && [ "$COMMAND" != "stop" ] && [ "$COMMAND" != "pid" ] && [ "$COMMAND" != "kill" ]; then
        echo "Invalid command: ${COMMAND}"
        echo "Available commands: start, log, stop, pid, kill"
        exit 1
else
        echo "Running command ${COMMAND} on program ${PROGRAM}"
fi

if [ "$PROGRAM" = "kafka" ]; then
        if [ "$COMMAND" = "start" ]; then
                kafkaStart
        elif [ "$COMMAND" = "stop" ]; then
                kafkaStop
        elif [ "$COMMAND" = "log" ]; then
                kafkaLog
        elif [ "$COMMAND" = "pid" ]; then
                kafkaPid
        elif [ "$COMMAND" = "kill" ]; then
                kafkaKill
        fi
fi

```

# 8 user .bashrc global variables and shortcut settings

alias of kafka scripts

    export KFK_SCRIPTS="/app/kafka4/scripts"
    alias kafka="${KFK_SCRIPTS}/kafka.sh kafka"

set jdk to path

    export JAVA_HOME=/app/jdk
    export PATH=$PATH:$JAVA_HOME/bin

set kafka heap options

    export KAFKA_HEAP_OPTS="-Xmx4096M -Xms512M"

# 9 Define as a system service

    [Unit]
    Description=Kafka Service
    
    [Service]
    Type=forking
    User=appuser
    ExecStart=/app/kafka4/scripts/kafka-service-start.sh --no-daemon
    Restart=always
    RestartSec=60
    TimeoutStopSec=60
    TimeoutStartSec=60
    
    [Install]
    WantedBy=default.target


# 10 start kafka using alias

kafka start

    [2025-05-05 13:02:46,436] INFO [BrokerServer id=2] Waiting for the broker to be unfenced (kafka.server.BrokerServer)
    [2025-05-05 13:02:46,473] INFO [BrokerLifecycleManager id=2] The broker has been unfenced. Transitioning from RECOVERY to RUNNING. (kafka.server.BrokerLifecycleManager)
    [2025-05-05 13:02:46,473] INFO [BrokerServer id=2] Finished waiting for the broker to be unfenced (kafka.server.BrokerServer)
    [2025-05-05 13:02:46,473] INFO authorizerStart completed for endpoint PLAINTEXT. Endpoint is now READY. (org.apache.kafka.server.network.EndpointReadyFutures)
    [2025-05-05 13:02:46,473] INFO [SocketServer listenerType=BROKER, nodeId=2] Enabling request processing. (kafka.network.SocketServer)
    [2025-05-05 13:02:46,473] INFO Awaiting socket connections on localhost:19092. (kafka.network.DataPlaneAcceptor)
    [2025-05-05 13:02:46,474] INFO [BrokerServer id=2] Waiting for all of the authorizer futures to be completed (kafka.server.BrokerServer)
    [2025-05-05 13:02:46,474] INFO [BrokerServer id=2] Finished waiting for all of the authorizer futures to be completed (kafka.server.BrokerServer)
    [2025-05-05 13:02:46,474] INFO [BrokerServer id=2] Waiting for all of the SocketServer Acceptors to be started (kafka.server.BrokerServer)
    [2025-05-05 13:02:46,474] INFO [BrokerServer id=2] Finished waiting for all of the SocketServer Acceptors to be started (kafka.server.BrokerServer)
    [2025-05-05 13:02:46,474] INFO [BrokerServer id=2] Transition from STARTING to STARTED (kafka.server.BrokerServer)
    [2025-05-05 13:02:46,474] INFO Kafka version: 4.0.0 (org.apache.kafka.common.utils.AppInfoParser)
    [2025-05-05 13:02:46,474] INFO Kafka commitId: 985bc99521dd22bb (org.apache.kafka.common.utils.AppInfoParser)
    [2025-05-05 13:02:46,474] INFO Kafka startTimeMs: 1746439366474 (org.apache.kafka.common.utils.AppInfoParser)
    [2025-05-05 13:02:46,475] INFO [KafkaRaftServer nodeId=2] Kafka Server started (kafka.server.KafkaRaftServer

kafka pid

    Running command pid on program kafka
    Kafka PID = 32359

kafka stop

    [2025-05-05 13:09:19,396] INFO Node to controller channel manager for registration shutdown (kafka.server.NodeToControllerChannelManagerImpl)
    [2025-05-05 13:09:19,397] INFO [ControllerRegistrationManager id=3 incarnation=OqC7on52Seayj2LYjQZcDw] closed event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,397] INFO [SocketServer listenerType=CONTROLLER, nodeId=3] Stopping socket server request processors (kafka.network.SocketServer)
    [2025-05-05 13:09:19,399] INFO [SocketServer listenerType=CONTROLLER, nodeId=3] Stopped socket server request processors (kafka.network.SocketServer)
    [2025-05-05 13:09:19,399] INFO [QuorumController id=3] QuorumController#beginShutdown: shutting down event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,399] INFO [SocketServer listenerType=CONTROLLER, nodeId=3] Shutting down socket server (kafka.network.SocketServer)
    [2025-05-05 13:09:19,401] INFO [SocketServer listenerType=CONTROLLER, nodeId=3] Shutdown completed (kafka.network.SocketServer)
    [2025-05-05 13:09:19,401] INFO [data-plane Kafka Request Handler on Broker 3], shutting down (kafka.server.KafkaRequestHandlerPool)
    [2025-05-05 13:09:19,401] INFO [data-plane Kafka Request Handler on Broker 3], shut down completely (kafka.server.KafkaRequestHandlerPool)
    [2025-05-05 13:09:19,401] INFO [ExpirationReaper-0-null]: Shutting down (org.apache.kafka.server.purgatory.DelayedOperationPurgatory$ExpiredOperationReaper)
    [2025-05-05 13:09:19,402] INFO [ExpirationReaper-0-null]: Stopped (org.apache.kafka.server.purgatory.DelayedOperationPurgatory$ExpiredOperationReaper)
    [2025-05-05 13:09:19,402] INFO [ExpirationReaper-0-null]: Shutdown completed (org.apache.kafka.server.purgatory.DelayedOperationPurgatory$ExpiredOperationReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Fetch]: Shutting down (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Fetch]: Stopped (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Fetch]: Shutdown completed (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Produce]: Shutting down (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Produce]: Shutdown completed (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Produce]: Stopped (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Request]: Shutting down (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Request]: Stopped (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-Request]: Shutdown completed (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-ControllerMutation]: Shutting down (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-ControllerMutation]: Shutdown completed (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [controller-3-ThrottledChannelReaper-ControllerMutation]: Stopped (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
    [2025-05-05 13:09:19,402] INFO [QuorumController id=3] closed event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,402] INFO [SharedServer id=3] Stopping SharedServer (kafka.server.SharedServer)
    [2025-05-05 13:09:19,402] INFO [MetadataLoader id=3] beginShutdown: shutting down event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,403] INFO [SnapshotGenerator id=3] close: shutting down event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,403] INFO [SnapshotGenerator id=3] closed event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,403] INFO [MetadataLoader id=3] closed event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,403] INFO [SnapshotGenerator id=3] closed event queue. (org.apache.kafka.queue.KafkaEventQueue)
    [2025-05-05 13:09:19,403] INFO Metrics scheduler closed (org.apache.kafka.common.metrics.Metrics)
    [2025-05-05 13:09:19,403] INFO Closing reporter org.apache.kafka.common.metrics.JmxReporter (org.apache.kafka.common.metrics.Metrics)
    [2025-05-05 13:09:19,403] INFO Metrics reporters closed (org.apache.kafka.common.metrics.Metrics)
    [2025-05-05 13:09:19,403] INFO App info kafka.server for 3 unregistered (org.apache.kafka.common.utils.AppInfoParser)
    [2025-05-05 13:09:19,403] INFO App info kafka.server for 3 unregistered (org.apache.kafka.common.utils.AppInfoParser)

# 11 Server alarm log error codes

    The operation perspective, tail the logs and match the any of following codes (INVALID_CLUSTER_ID,UNKNOWN_LEADER_EPOCH..) and then alarm is executed.
    
    INVALID_CLUSTER_ID: The request indicates a clusterId which does not match the value cached in meta.properties.
    
    FENCED_LEADER_EPOCH: The leader epoch in the request is smaller than the latest known to the recipient of the request.
    
    UNKNOWN_LEADER_EPOCH: The leader epoch in the request is larger than expected. Note that this is an unexpected error. Unlike normal Kafka log replication, it cannot happen that the follower receives the newer epoch before the leader.
    
    OFFSET_OUT_OF_RANGE: Used in the Fetch API to indicate that the follower has fetched from an invalid offset and should truncate to the offset/epoch indicated in the response.
    
    NOT_LEADER_FOR_PARTITION: Used in DescribeQuorum and AlterPartitionReassignments to indicate that the recipient of the request is not the current leader.
    
    INVALID_QUORUM_STATE: This error code is reserved for cases when a request conflicts with the local known state. For example, if two separate nodes try to become leader in the same epoch, then it indicates an illegal state change.
    
    INCONSISTENT_VOTER_SET: Used when the request contains inconsistent membership.


## Port & LAG check scripts

    Check the 9092, 19092 ports, periodically. 
    Check topics of LAG periodically 

# 12 References

    https://github.com/inomera
    https://kafka.apache.org/downloads
    https://kafka.apache.org/blog#apache_kafka_400_release_announcement
    https://cwiki.apache.org/confluence/display/KAFKA/KIP-896%3A+Remove+old+client+protocol+API+versions+in+Kafka+4.0
