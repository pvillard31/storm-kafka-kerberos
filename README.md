# Storm topology - Consume from Kafka and publish to Kafka - with Kerberos

Environment with [Hortonworks](https://hortonworks.com/) distribution:
- Cluster deployed with [Ambari 2.5.1.0](https://docs.hortonworks.com/HDPDocuments/Ambari/Ambari-2.5.1.0/index.html), [HDP 2.6.1](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.1/index.html) and [HDF 3.0.0](https://docs.hortonworks.com/HDPDocuments/HDF3/HDF-3.0.0/index.html)
- Cluster is Kerberized against an Active Directory
- Storm is deployed on 3 nodes, Kafka is deployed on 3 nodes, and a 3-nodes Zookeeper quorum
- Authorizations for Kafka are managed by Ranger

Versions are Storm 1.1.0 and Kafka 0.10.1

Objective is to have a Storm topology consuming from a Kafka topic ("inputTopicStorm") using KafkaSpout, to send the data into a bolt that will not do anything (IdentityBolt) and send back the data into another Kafka topic ("outputTopicStorm") using KafkaBolt.

Starting with Kafka client 0.10.2 and [KAFKA-4259](https://issues.apache.org/jira/browse/KAFKA-4259), it is possible to dynamically define the JAAS configuration to connect to the Kafka topic. It means that, in theory, we could use a specific user to consume the data from the Kafka topic, and use another one to publish the data. It also means that it's not necessary to rely in JAAS configuration file anymore.

To build:

````
mvn clean package
````

To submit the topology:

````
[pvillard@storm-1 tmp]$ storm jar kafka-storm-kafka-0.0.1-SNAPSHOT.jar example.KafkaStormKafkaTopology
````

### Code explanations

I define the properties I want to use in my Storm topology:
(note that the keytab need to be available on all the Storm nodes)

````java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka-1.example.com:6667,kafka-2.example.com:6667,kafka-3.example.com:6667");
props.put("acks", "1");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("security.protocol", "SASL_PLAINTEXT");
props.put("sasl.jaas.config", "com.sun.security.auth.module.Krb5LoginModule required "
            + "useTicketCache=false "
            + "renewTicket=true "
            + "serviceName=\"kafka\" "
            + "useKeyTab=true "
            + "keyTab=\"/home/pvillard/pvillard.keytab\" "
            + "principal=\"pvillard@EXAMPLE.COM\";");
````

#### Kafka Spout

````java
// Kafka spout getting data from "inputTopicStorm"
KafkaSpoutConfig<String, String> kafkaSpoutConfig = KafkaSpoutConfig
        .builder(props.getProperty("bootstrap.servers"), "inputTopicStorm")
        .setGroupId("storm")
        .setProp(props)
        .setRecordTranslator((r) -> new Values(r.topic(), r.key(), r.value()), new Fields("topic", "key", "message"))
        .build();
KafkaSpout<String, String> kafkaSpout = new KafkaSpout<>(kafkaSpoutConfig);
````

#### Kafka Bolt

````java
// Kafka bolt to send data into "outputTopicStorm"
KafkaBolt<String, String> kafkaBolt = new KafkaBolt<String, String>()
        .withProducerProperties(props)
        .withTopicSelector(new DefaultTopicSelector("outputTopicStorm"))
        .withTupleToKafkaMapper(new FieldNameBasedTupleToKafkaMapper<String, String>());
````

### Notes

If you have a blank page when accessing Storm UI from Ambari, you'll need to add the following in "Custom storm-site":
> ui.header.buffer.bytes=65536

I also added:
> supervisor.run.worker.as.user=true

To be able to use the Storm CLI with Kerberos, I did:
````
[pvillard@storm-1 ~]# cd
[pvillard@storm-1 ~]# mkdir .storm
[pvillard@storm-1 ~]# vi .storm/storm.yaml
````

And added the following:

````properties
nimbus.seeds: ['storm-1.example.com','storm-2.example.com','storm-3.example.com']
nimbus.thrift.port: 6627
java.security.auth.login.config: "/etc/storm/conf/client_jaas.conf"
storm.thrift.transport: "org.apache.storm.security.auth.kerberos.KerberosSaslTransportPlugin"
````
You can find the values for nimbus.seeds & nimbus.thrift.port in /etc/storm/conf/storm.yaml

Then you can check if you can use the Storm CLI executing:
````
[pvillard@storm-1 ~]# storm list
````
