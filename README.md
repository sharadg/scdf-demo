# spring-cloud-dataflow-demo

## Spin up backing services
---
- Running RabbitMQ locally using Docker
  ```docker run -d -p 5672:5672 -p 15672:15672 --name rabbitmq --hostname local-rabbit rabbitmq:3.7.4-management```
  
- Running MySQL locally using Docker
  ```docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=admin -e MYSQL_DATABASE=scdf -p 3306:3306 -d mysql:latest```
  
- Running Redis locally using Docker
  ```docker run -d --name redis -p 6379:6379 redis```

```
docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED              STATUS              PORTS                                                                                        NAMES
f0e9e090466b        redis                       "docker-entrypoint.s…"   2 seconds ago        Up 2 seconds        0.0.0.0:6379->6379/tcp                                                                       redis
2bda42708880        mysql:latest                "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:3306->3306/tcp                                                                       mysql
5fea4b632d32        rabbitmq:3.7.4-management   "docker-entrypoint.s…"   4 minutes ago        Up 4 minutes        4369/tcp, 5671/tcp, 0.0.0.0:5672->5672/tcp, 15671/tcp, 25672/tcp, 0.0.0.0:15672->15672/tcp   rabbitmq
```

###########################
Start SCDF server and shell
###########################
---
- Run SCDF server locally with mysql and rabbitmq info
  java -jar spring-cloud-dataflow-server-local-1.4.0.RELEASE.jar --spring.datasource.url=jdbc:mysql://localhost:3306/scdf --spring.datasource.username=root --spring.datasource.password=admin --spring.datasource.driver-class-name=org.mariadb.jdbc.Driver --spring.rabbitmq.host=127.0.0.1 --spring.rabbitmq.username=guest --spring.rabbitmq.password=guest --spring.rabbitmq.port=5672
  
- Run the SCDF Shell
  java -jar spring-cloud-dataflow-shell-1.4.0.RELEASE.jar
  
- Import the SCSt App Starters
  app import --uri http://bit.ly/Celsius-SR1-stream-applications-rabbit-maven
- Confirm its working: 
  app list
  app info sink:file

###########################
Simple Streams
###########################
---
### Simple HTTP and Log stream
- Create a SCDF Stream definition, deploy it.
  stream create --definition "http --port=8086 | file --directory=/Users/sgupta/development/pivotal/scdf/tmp --suffix=txt --name=output" --name hello
  stream list
  stream deploy --name hello
  
- Stream is running. Test it!
  http POST :8086 hello=world
  HTTP/1.1 202 
  Content-Length: 0
  Date: Thu, 05 Apr 2018 18:58:24 GMT
  X-Application-Context: http-source:8086
  
  cat output.txt 
  {"hello": "world"}
  
### Simple HTTP, Transform and Log stream
- Create & deploy
  http --port=8888 | transform --expression="payload.toUpperCase()" | log
- Test it
  http -v POST :8888 hello=world marvel=avengers
  
###########################
Simple Tasks, Batch Job
###########################
---
### Simple Task  
- Import Task App Starters
  app import --uri http://bit.ly/Clark-GA-task-applications-maven
  
- Create Task definition, launch it
  task create --name demotask --definition "timestamp --format='yyyy-MM-dd'"
  task list
  task launch --name demotask
  task execution list

### Batch File Ingestion
- Import fileIngest App Starters
  app register --type task --name fileIngest --uri file:///Users/sgupta/development/pivotal/scdf/spring-cloud-dataflow-samples/batch/file-ingest/target/ingest-1.0.0.jar
  
- Create Task definition, launch it
  task create fileIngestTask --definition fileIngest
  task list
  task launch fileIngestTask --arguments "filePath=file:///Users/sgupta/development/pivotal/scdf/spring-cloud-dataflow-samples/batch/file-ingest/src/main/resources/data.csv"
  task execution list

###########################
Twitter Stream (Shows Analytics)
###########################
---
### Twitter Analytics
- Deploy the following streams
  tweets = twitterstream --access-token-secret=f4Q5TOBcoS5DNBqniwrF1VBRCjmj5ByqTz0DchFL9L0AK --access-token=51016536-o39amDGsaSfsASPd1Jg863KEWixhhvEsuv7Wp6UOS --consumer-secret=sXSqxORFJjsMVeUfHO0hA37jKWmBHRVdt1ZDxdp0XbokiYlIuo --consumer-key=L3kKDwfvC4ebqamys4FuB05Nd | log
  :tweets.twitterstream > field-value-counter --field-name=lang --name=language
  :tweets.twitterstream > field-value-counter --name=hashtags --field-name=entities.hashtags.text
- Deploy and check the Analytics tab

###########################
Object Detection (AI/DL)
###########################
---
### Register image-viewer sink and object-detection processor application
- Image Viewer
  app register --name image-viewer --type sink --uri file:///Users/sgupta/development/pivotal/scdf/image-processing/image-viewer/target/image-viewer-0.0.1-SNAPSHOT.jar
- Object Detection (TensorFlow)
  app register --name object-detector --type sink --uri file:///Users/sgupta/development/pivotal/scdf/tensorflow/apps/object-detection-processor-rabbit/target/object-detection-processor-rabbit-2.0.0.BUILD-SNAPSHOT.jar
- Create Stream & deploy
  stream create --name object-detection --definition "file --directory='/Users/sgupta/development/pivotal/scdf/object-detection/input' | object-detector --mode=header | image-viewer"
  stream deploy --name object-detection
  
###########################
FTP Stream
###########################
---
### Start Docker based FTP server
- FTP Container (optionally: bash into container to read the logs)
  docker run -d --name ftpd_server -p 21:21 -p 30000-30009:30000-30009 -e "PUBLICHOST=localhost" -e "ADDED_FLAGS=-d -d -O w3c:/var/log/pure-ftpd/transfer.log" stilliard/pure-ftpd:latest
- Optional: bash into container to read the logs or create new user)
  docker exec -it ftpd_server /bin/bash
- Create an ftp user:
  pure-pw useradd bob -f /etc/pure-ftpd/passwd/pureftpd.passwd -m -u ftpuser -d /home/ftpusers/bob
  (password: bob)
- Connect to FTP from local machine & upload a file for testing
  lftp localhost -p 21 -u bob,bob
  
### Create & deploy FTP stream
- Create FTP stream
  simple_ftp = ftp --local-dir=/Users/sgupta/development/pivotal/scdf/tmp --remote-dir=/ --username=bob --password=bob --host=localhost --filename-pattern=cf* --fixed-delay=5 --time-unit=MINUTES > :file_collector
  log_ftp = :file_collector | log

###########################
HTTP & JDBC
###########################
- Create MySQL database and tables
  mysqlsh root@localhost -P 3306 --password=admin --database=test --sql
  create database test;
  CREATE TABLE `names` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `name` varchar(255) DEFAULT NULL,
    `city` varchar(50) DEFAULT NULL,
    `street` varchar(50) DEFAULT NULL,
    PRIMARY KEY (`id`)
  )
  
- Create HTTP & JDBC
  http_jdbc = http --server.port=8787 | jdbc --table-name=names --columns=name,city,street --password=admin --username=root --driver-class-name=org.mariadb.jdbc.Driver --url='jdbc:mysql://localhost:3306/test'
  http -v POST :8787 name=bruce_lee city=shanghai street=lee_blvd
  http -v POST :8787 < ../ftp_jdbc_demo/cf-cities.json

- Alternatively, make the same file available on FTP server
- Create Stream & Test
  ftp_jdbc = ftp --local-dir='/Users/sgupta/development/pivotal/scdf/tmp' --remote-dir='/' --username=bob --password=bob --host=localhost --filename-pattern=cf* --fixed-delay=2 --time-unit=MINUTES --mode=lines --with-markers=false --delete-remote-files=true | jdbc_2 --table-name=names --columns=name,city,street --password=admin --username=root --driver-class-name=org.mariadb.jdbc.Driver --url='jdbc:mysql://localhost:3306/test'
  
