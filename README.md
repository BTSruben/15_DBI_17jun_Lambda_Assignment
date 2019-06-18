**Schema**

```bash
MEETUP <<< SENSOR >>> KAFKA <<< SPARK >>>> HDFS
                   kafka:9092          hdfs://namenode:8020
                 zookeeper:2181
```

- Live stream meetup data from
  
[https://www.meetup.com/meetup_api/docs/stream/2/rsvps/#websockets](https://www.meetup.com/meetup_api/docs/stream/2/rsvps/#websockets)

- Alternative ssh using docker

```bash
    docker run -it -v <ABSOLUTE_PATH_KEYPAIR>:/mykey.pem --rm kroniak/ssh-client bash
    chmod 400 /mykey.pem
    ssh -i /mykey.pem <user@hostname>
```

**Steps of the Assignment**

-Launch AWS Instance Command

```bash
    aws ec2 run-instances --image-id ami-0ebb3a801d5fb8b9b --count 1 --instance-type m5.xlarge --key-name <keypair> --security-group-ids sg-0e8cb59d207ca3ed3 --subnet-id subnet-0ba219ffbd8c264d2 --associate-public-ip-address
```

- Describe AWS Instance Command

```bash
    aws ec2 describe-instances --filter "Name=instance-id,Values=<id-instance>"
```

- Terminate AWS Instance Command

```bash
    aws ec2 terminate-instances --instance-ids <id-instance>
```

**Installing Docker in Amazon Linux**

- Connecting to instance

```bash
    ssh -i <keypair> <ec2-user@hostname>
```

- Install software

```bash
    sudo yum update -y
```

```bash
    sudo amazon-linux-extras install docker
```

```bash
    sudo service docker start
```

```bash
    sudo usermod -a -G docker ec2-user
```

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

```bash
sudo chmod +x /usr/local/bin/docker-compose
```

```bash
    sudo yum install git
```

- **Logout and Login again using ssh**

- Git clone our files

```bash
    git clone https://github.com/BTSruben/15_DBI_17jun_Lambda_Assignment.git
    mv 15_DBI_17jun_Lambda_Assignment/* .
    mkdir spark_docker/data
    ./start-docker-compose.sh
```
- Check if you have all the architecture up. See the schema, above.

```bash
    docker ps
```

- Exec Kafka Producer to add our data to KAFKA

```bash
    docker exec -it sensor sh -c 'curl -i http://stream.meetup.com/2/rsvps | kafkacat -b kafka:9092 -t stream' &
```
- You can check if data is arriving to KAFKA ( Carefull!! When you cancel this process, the above step will stop you can use command: bg or send command again )

```bash
    docker exec -it sensor sh -c 'kafkacat -b kafka:9092 -t stream'
```

- Launch SparkShell, our Kafka Consumer

```bash
    docker exec -it spark  spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.0,org.apache.kafka:kafka-clients:2.2.0,org.apache.spark:spark-tags_2.11:2.4.0,org.apache.spark:spark-sql_2.11:2.4.0
```

- Copy all code to send data from kafka to Hadoop System

```bash
import java.sql.Timestamp
case class VenueModel(venue_name: Option[String], lon: Option[Double], lat: Option[Double], venue_id: Option[String])
case class MemberModel(member_id: Long, photo: Option[String], member_name: Option[String])
case class Event(event_name: String, event_id: String, time: Long, event_url: Option[String])
case class GTopicModel(urlkey: String, topic_name: String)
case class GroupModel(group_topics: Array[GTopicModel], group_city: String, group_country: String, group_id: Long, group_name: String, group_lon: Double, group_urlname: String, group_state: Option[String], group_lat: Double)
case class MeetupModel(venue: VenueModel, visibility: String, response: String, guests: Long, member: MemberModel, rsvp_id: Long,  mtime: Long, group: GroupModel)


val dstream = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "kafka:9092").option("subscribe", "stream").load().selectExpr("CAST(value AS STRING)").as[String]

import org.json4s._
import org.json4s.jackson.JsonMethods._
val dsMeetups = dstream.map(r=> { implicit val formats = DefaultFormats; parse(r).extract[MeetupModel] } )
val query = dsMeetups.flatMap(meetup=>meetup.group.group_topics)

import org.apache.spark.sql.streaming.{OutputMode}
query.writeStream.format("parquet").outputMode(OutputMode.Append()).option("checkpointLocation", "/tmp").option("path", "hdfs://namenode:8020/spark").start()
```
- Open another terminal, login to the instance and login to our hadoop namenode container.

```bash
    ssh -i <keypair> <ec2-user@hostname>
    docker exec -it namenode bash
```

- Spark is sending data to the path: hdfs://namenode:8020/spark. If you want to check the path inside namenode container:

```bash
    hdfs dfs -ls /spark
```
- You can check the content of your data using this command:

```bash
parquet-tools cat --json hdfs://namenode:8020/<path/filename>
```

**WHEN YOU HAVE FINISHED, TERMINATE THE INSTANCE!!**

```bash
    aws ec2 terminate-instances --instance-ids <id-instance>
```

