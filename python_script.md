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



