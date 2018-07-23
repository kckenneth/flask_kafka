|Title |  Kafka deployment in Web-based Mobile Game |
|-----------|----------------------------------|
|Author | Kenneth Chen |
|Utility | Flask, Kafka, Spark, HDFS, Droplet, Docker |
|Date | 7/1/2018 |

__Synopsis__  

We are now building a pipeline for smooth communications between web gamers. Intuitively, internet users communicate via sending and receiving messages (data packets) across the web. The protocol that facilitates data transfer is HTTP with its conventional GET, POST method. Adding on top of the HTTP for reliable data transfer is TCP protocol (more advanced algorithm such as TCP Reno). Here we are mainly concerned with our customers' gaming activity and their web-based communication to our game servers. We will use Flask, a small web framework that will do all the jobs in the background for us to send and receive web-based messages generated by kafka. In addition, all the gamer activities or events will be relayed to our backend server by kafka and stored in our hadoop file system. A data flow chart was shown below.   

<p align="center">
<img src="img/flowchart.png" width="600"></p>
<p align="center">Figure 1. Data transfer in web-based communication</p>

__Procedure__  

In mobile game community, web-based communication is a crucial part of the game development in which players response (events) to game play must be instantly updated in order for players to enjoy playing a game without any glitch. We deployed **Flask** for our web-based mobile game called **"Build a Nation"** and used **Kafka** to manage gamer web activities such as "purchase a sword", "purchase a horse", "purchase a shield" etc etc etc. 

I laid out a step by step implementation of Flask and Kafka in details as follows.  

## In Droplet, Update images 
### 1. Updating Docker Images 
```
docker pull midsw205/base:0.1.8
docker pull confluentinc/cp-zookeeper:latest
docker pull confluentinc/cp-kafka:latest
docker pull midsw205/spark-python:0.0.5
docker pull midsw205/cdh-minimal:latest
```
- midsw205/base \t [python, jupyter apps]  
- confluentinc/cp-zookeeper \t [zookeeper manager for kafka]  
- confluentinc/cp-kafka [kafka app]
- midsw205/spark-python

### 2. Logging into the assignment folder
```
cd w205/assignment-10-kckenneth/
```

### 3. Checking what's in my directory 
```
ls  
```

### 4. Making sure at which branch I am on git
```
git branch   
```
### 5. Checking if there's any pre-existing docker containers running
```
docker-compose ps  
docker ps -a  
```

##### if need be, remove any running containers by rm
```
docker rm -f $(docker ps -aq) 
```
### 6. Run a single docker container midsw205 in bash mode
```
docker run -it --rm -v /home/science/w205:/w205 midsw205/base:latest bash
```
## Inside the Docker Container
1. check into assignment 10 folder,
2. check git branch, create assignment branch if necessary  
3. create docker-compose.yml with 4 containers
  - zookeeper  
  - kafka  
  - mids  
  - spark
4. create game_api.py

```
cd assignment-10-kckenneth  
ls  
git status  
git branch 
git checkout -b assignment  
vi docker-compose.yml  
vi game_api.py
exit  
```

## Flask game_api.py in details  
#### I introduced 3 more game actions: purchase a shield, upgrade a sword and upgrade a shield

```
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask, request

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')


def log_to_kafka(topic, event):
    event.update(request.headers)
    producer.send(topic, json.dumps(event).encode())


@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "\nSword Purchased!\n"
    
    
@app.route("/purchase_a_shield")
def purchase_a_shield():
    purchase_shield_event = {'event_type': 'purchase_shield'}
    log_to_kafka('events', purchase_shield_event)
    return "\nShield Purchased!\n"
    
    
@app.route("/upgrade_a_sword")
def upgrade_a_sword():
    upgrade_sword_event = {'event_type': 'upgrade_sword'}
    log_to_kafka('events', upgrade_sword_event)
    return "\nSword Upgraded!\n"
    
    
@app.route("/upgrade_a_shield")
def upgrade_a_shield():
    upgrade_shield_event = {'event_type': 'upgrade_shield'}
    log_to_kafka('events', upgrade_shield_event)
    return "\nShield Upgraded!\n"
```
 
Here we are using game_api.py to call flask module to run. Any messages or events that game players generate from the web such as "purchase a sword", "purchase a horse", "upgrade the arrow", etc etc etc will be fed into Kafka. Therefore we import the **KafkaProducer** library in our game_app.py script. Kafka will then publish those events in json format. In the frontend, we can enrich our web page with more **CSS** features for gamer experiences. But in this exercise, we will only execute a simple message. In the backend, those messages will be stored and analyzed in our game server (**Data Analytics**). For example, "purchase a sword" events will be stored with its game player ID in our game server. Any game activities that a game player can now pursue or execute because of the possession of a sword will be relayed from the backend game server to the front end so that the player could now execute additional activity in the game due to the possession of the sword.  

#### One of the flask app methods, purchase_sword(), to remind myself
```
@app.route("/purchase_a_sword")                                   # gamer input "purchase_a_sword" after the slash /
def purchase_sword():
    event_logger.send(events_topic, 'purchased_sword'.encode())   # "purchase_a_sword" event will be recorded as "purchased_sword" in Kafka. It will be published instantly and consumed at the backend 
    return "\nSword Purchased!\n"                                 # gamer received "Sword Purchased!" screen display
```

## In Droplet, I spin up the cluster in detached mode by -d
```
docker-compose up -d
```
### Check if the zookeeper is up and running by finding the *binding* word in the logs file
```
docker-compose logs zookeeper | grep -i binding  
```
### I also checked kafka is up and running by searching the word *started*
```
docker-compose logs kafka | grep -i started
```
## I. Kafka 1st step -- Create a Topic
#### I created a topic *events* with partition 1, replication-factor 1
```
docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
```
#### I checked the broker I just created by *describe* function  
```
docker-compose exec kafka kafka-topics --describe --topic events --zookeeper zookeeper:32181
```  
## II. Kafka 2nd step -- Produce Messages 
#### Kafka messages will be from gamers web activities. Since there are two ends in this exercise, we need to open two CLI windows. 

<p align="center">
<img src="img/hadoop.png" width="600"></p>
<p align="center">Figure 2. Kafka and Flask in Game activity management</p>

- Gamer window  
- Flask window  

There are two processes.  
1) We first run the flask in flask window in order to initiate a web-based application such as **"Build A Nation"** game in this case.  
2) A gamer will execute game activites on the web browser. Here "purchase a sword" action will be executed. We will therefore execute from the gamer CLI window. 

### 1) Run python flask in Flask Window
```
docker-compose exec mids env FLASK_APP=/w205/assignment-10-kckenneth/game_api.py flask run
```
### 2) Gamer Activity in Gamer Window 

##### You need to ssh into Droplet from another CLI window. Once you're in the Droplet, go to /w205/assignment-10-kckenneth/ folder
```
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
docker-compose exec mids curl http://localhost:5000/
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
docker-compose exec mids curl http://localhost:5000/purchase_a_shield
docker-compose exec mids curl http://localhost:5000/upgrade_a_sword
docker-compose exec mids curl http://localhost:5000/purchase_a_shield
docker-compose exec mids curl http://localhost:5000/upgrade_a_shield
docker-compose exec mids curl http://localhost:5000/
docker-compose exec mids curl http://localhost:5000/purchase_a_shield
```

## III. Kafka 3rd step -- Consume Game Events or Messages
- 1) We consume Kafka message at the backend. So it's supposed to be in another CLI window unless we want to stop the Flask app.  
- 2) We can also consume Kafka message in Gamer CLI window. However, in reality, gamer CLI window wouldn't have the docker cluster at all. Remember that it's for the convenience.  

### Consume game events or messages in `mids` container
```
docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t events -o beginning -e"
```
#### 10 messages are consumed  
```
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "upgrade_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "upgrade_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
% Reached end of topic events [0] at offset 10: exiting
```

#### Consume game events in `pyspark` container
1. We first launch the `pyspark` container  
```
docker-compose exec spark pyspark
```
2. We then consume the events or messages in spark  
```
>>> raw_events = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","events").option("startingOffsets", "earliest").option("endingOffsets", "latest").load() 
>>> raw_events.cache()
>>> raw_events.show()
+----+--------------------+------+---------+------+--------------------+-------------+
| key|               value| topic|partition|offset|           timestamp|timestampType|
+----+--------------------+------+---------+------+--------------------+-------------+
|null|[7B 22 48 6F 73 7...|events|        0|     0|2018-07-22 21:42:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     1|2018-07-22 21:42:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     2|2018-07-22 21:42:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     3|2018-07-22 21:42:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     4|2018-07-22 21:42:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     5|2018-07-22 21:42:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     6|2018-07-22 21:43:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     7|2018-07-22 21:43:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     8|2018-07-22 21:43:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     9|2018-07-22 21:43:...|            0|
+----+--------------------+------+---------+------+--------------------+-------------+
```
Since the events we just consumed are binary format in spark (written in scala programming language), we transformed our data into `string` format in pyspark. 

```
>>> events = raw_events.select(raw_events.value.cast('string'))
>>> events.printSchema()
root
 |-- value: string (nullable = true)

>>> events.show(20, False)

+---------------------------------------------------------------------------------------------------------+
|value                                                                                                    |
+---------------------------------------------------------------------------------------------------------+
|{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"} |
|{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"} |
|{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}        |
|{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"} |
|{"Host": "localhost:5000", "event_type": "purchase_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"}|
|{"Host": "localhost:5000", "event_type": "upgrade_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"} |
|{"Host": "localhost:5000", "event_type": "upgrade_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}  |
|{"Host": "localhost:5000", "event_type": "purchase_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"}|
|{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}        |
|{"Host": "localhost:5000", "event_type": "purchase_shield", "Accept": "*/*", "User-Agent": "curl/7.47.0"}|
+---------------------------------------------------------------------------------------------------------+
```

3. Since the kafka messages are in json format, we will load the messages into json format in pyspark  
```
>>> import json
>>> extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()
>>> extracted_events.printSchema()
root
 |-- Accept: string (nullable = true)
 |-- Host: string (nullable = true)
 |-- User-Agent: string (nullable = true)
 |-- event_type: string (nullable = true)
 
>>> extracted_events.show()
+------+--------------+-----------+---------------+
|Accept|Host          |User-Agent |event_type     |
+------+--------------+-----------+---------------+
|*/*   |localhost:5000|curl/7.47.0|purchase_sword |
|*/*   |localhost:5000|curl/7.47.0|purchase_sword |
|*/*   |localhost:5000|curl/7.47.0|default        |
|*/*   |localhost:5000|curl/7.47.0|purchase_sword |
|*/*   |localhost:5000|curl/7.47.0|purchase_shield|
|*/*   |localhost:5000|curl/7.47.0|upgrade_shield |
|*/*   |localhost:5000|curl/7.47.0|upgrade_sword  |
|*/*   |localhost:5000|curl/7.47.0|purchase_shield|
|*/*   |localhost:5000|curl/7.47.0|default        |
|*/*   |localhost:5000|curl/7.47.0|purchase_shield|
+------+--------------+-----------+---------------+
```

# Gamer Activities Analytics in Spark

As a preliminary, I analyzed gamer activities in spark sql environment. I first registered the table extracted in json format into `gamer` table. I then counted the number of unique activities that gamers pursued and listed them. 

```
>>> extracted_events.registerTempTable('gamer')
>>> spark.sql("SELECT event_type, COUNT(event_type) as event_count FROM gamer GROUP BY event_type ORDER BY event_count DESC").show()
+---------------+-----------+                                                   
|     event_type|event_count|
+---------------+-----------+
|purchase_shield|          3|
| purchase_sword|          3|
|        default|          2|
| upgrade_shield|          1|
|  upgrade_sword|          1|
+---------------+-----------+
```
We found that there are more activities on `purchase_sword` and `purchase_shield` and upgrading activities are as few as one user per activity. 

## Exit
```
docker-compose down
```
# Summary
We developed our game **Build A Nation** with more game features such as 
- purchase a sword
- purchase a shield  
- upgrade a sword  
- upgrade a shield 

We launched our web-based game with micro webservice app in Python, `Flask`. Any gamer activities are recorded in `Kafka` and relayed to our server. We then consumed gamer activities or events in our backend. We wrote our script to produce more detailed events such as 
- Accept  
- Host  
- User-Agent  
- event_type  
we have more information on gamer activities. Since event messages were coded in json format, we consumed them pyspark and analyzed player activites. In the following week, we will develop more game features and data analytics. 
