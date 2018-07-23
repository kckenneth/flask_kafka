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


