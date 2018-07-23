# Python script walk-through

This is a detailed walk through on python script we're using in our web-based game development.  

# Flask Implementation
## game_api.py 

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
    
 
@app.route("/purchase_a_knife")
def purchase_a_knife():
    purchase_knife_event = {'event_type': 'purchase_knife',
                            'description': 'very sharp knife'}
    log_to_kafka('events', purchase_knife_event)
    return "Knife Purchased!\n"
```

#### I introduced 3 more game actions: purchase a shield, upgrade a sword and upgrade a shield  
 
Here we are using game_api.py to call flask module to run. Any messages or events that game players generate from the web such as "purchase a sword", "purchase a horse", "upgrade the arrow", etc etc etc will be fed into Kafka. Therefore we import the **KafkaProducer** library in our game_app.py script. Kafka will then publish those events in json format. In the frontend, we can enrich our web page with more **CSS** features for gamer experiences. But in this exercise, we will only execute a simple message. In the backend, those messages will be stored and analyzed in our game server (**Data Analytics**). For example, "purchase a sword" events will be stored with its game player ID in our game server. Any game activities that a game player can now pursue or execute because of the possession of a sword will be relayed from the backend game server to the front end so that the player could now execute additional activity in the game due to the possession of the sword.  

#### One of the flask app methods, purchase_sword(), to remind myself
```
@app.route("/purchase_a_sword")                                   # gamer input "purchase_a_sword" after the slash /
def purchase_sword():
    event_logger.send(events_topic, 'purchased_sword'.encode())   # "purchase_a_sword" event will be recorded as "purchased_sword" in Kafka. It will be published instantly and consumed at the backend 
    return "\nSword Purchased!\n"                                 # gamer received "Sword Purchased!" screen display
```

# Spark-submit 
## extract_events.py

```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs"""

import json
from pyspark.sql import SparkSession


def main():
    """main"""
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()
    
    # Reading kafka events into raw_events variable
    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()
    
    # Since all events are coded in binary format in spark, we change them into string format
    events = raw_events.select(raw_events.value.cast('string'))
    extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()

    # saving extracted events into HDFS 
    extracted_events \
        .write \
        .parquet("/tmp/extracted_events")


if __name__ == "__main__":
    main()
```

## transform_events.py
```
#!/usr/bin/env python
"""Extract events from kafka, transform, and write to hdfs"""

import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('string')
def munge_event(event_as_json):
    event = json.loads(event_as_json)
    event['Host'] = "moe" # silly change to show it works
    event['Cache-Control'] = "no-cache"
    return json.dumps(event)


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    # Reading kafka events into raw_events variable
    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    # Transforming binary into string format
    munged_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .withColumn('munged', munge_event('raw'))
    munged_events.show()

    extracted_events = munged_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.munged))) \
        .toDF()
    extracted_events.show()

    # Saving extracted events in HDFS
    extracted_events \
        .write \
        .mode("overwrite") \
        .parquet("/tmp/extracted_events")


if __name__ == "__main__":
    main()
```

## separate_events.py
```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('string')
def munge_event(event_as_json):
    event = json.loads(event_as_json)
    event['Host'] = "moe" # silly change to show it works
    event['Cache-Control'] = "no-cache"
    return json.dumps(event)


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    munged_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .withColumn('munged', munge_event('raw'))

    extracted_events = munged_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.munged))) \
        .toDF()

    sword_purchases = extracted_events \
        .filter(extracted_events.event_type == 'purchase_sword')
    sword_purchases.show()
    # sword_purchases \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/sword_purchases")

    default_hits = extracted_events \
        .filter(extracted_events.event_type == 'default')
    default_hits.show()
    # default_hits \
        # .write \
        # .mode("overwrite") \
        # .parquet("/tmp/default_hits")


if __name__ == "__main__":
    main()
```

# Extract, Transform, Filter, Save pipeline script `filtered_writes.py`

This script will do all the jobs of extracting kafka messages, encode them into 'string' format, transform them into json, and save into HDFS. 

```
#!/usr/bin/env python
"""Extract events from kafka and write them to hdfs
"""
import json
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import udf


@udf('boolean')
def is_purchase(event_as_json):
    event = json.loads(event_as_json)
    if event['event_type'] == 'purchase_sword':
        return True
    return False


def main():
    """main
    """
    spark = SparkSession \
        .builder \
        .appName("ExtractEventsJob") \
        .getOrCreate()

    raw_events = spark \
        .read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:29092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()

    purchase_events = raw_events \
        .select(raw_events.value.cast('string').alias('raw'),
                raw_events.timestamp.cast('string')) \
        .filter(is_purchase('raw'))

    extracted_purchase_events = purchase_events \
        .rdd \
        .map(lambda r: Row(timestamp=r.timestamp, **json.loads(r.raw))) \
        .toDF()
    extracted_purchase_events.printSchema()
    extracted_purchase_events.show()

    extracted_purchase_events \
        .write \
        .mode('overwrite') \
        .parquet('/tmp/purchases')


if __name__ == "__main__":
    main()
```


