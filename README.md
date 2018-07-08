|Title |  Kafka deployment in Web-based Mobile Game |
|-----------|----------------------------------|
|Author | Kenneth Chen |
|Utility | Kafka, Spark, HDFS, Droplet, Docker, LA |
|Date | 6/29/2018 |

__Procedure__  

In mobile game community, web-based communication is a crucial part of the game development in which players response (events) to game play must be instantly updated in order for players to continue the game without any glitch. We deployed Kafka for our web-based mobile game called "Build a Nation".  

I laid out a step by step implementation of Kafka in details in the following.  

## In Droplet, Update images 
### 1. Updating Docker Images 
```
docker pull midsw205/base:0.1.8
docker pull confluentinc/cp-zookeeper:latest
docker pull confluentinc/cp-kafka:latest
```
### 2. Logging into the assignment folder
```
cd w205/assignment-09-kckenneth/
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
1. check into assignment 9 folder,
2. check git branch, create assignment branch if necessary  
3. create docker-compose.yml with 3 containers
  - zookeeper  
  - kafka  
  - mids  
4. create game_app.py

```
cd assignment-09-kckenneth  
ls  
git status  
git branch 
git checkout -b assignment  
vi docker-compose.yml  
vi game_app.py
exit  
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
### Run python flask
```
docker-compose exec mids env FLASK_APP=/w205/flask-with-kafka/game_api.py flask run
```
### In another CLI window
```
docker-compose exec mids curl http://localhost:5000/
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```

## Kafka consumption 

Since we added kafka script in game_app.py that will publish player events (or action), we can consume the events log by kafkacat.  

```
docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t events -o beginning -e"
```

## Exit
```
docker-compose down
```

