# Overview
도커 컴포즈를 이용하여 카프카 테스트 환경을 만들어본다.

# ZOOKEEPER 설정
* zk-single-kafka-single.yml
~~~yaml
version: '3'

services:
    zoo1:
        image: zookeeper:3.4.9
        hostname: zoo1
        ports:
        - "2181:2181"
        environment:
          ZOO_MY_ID: 1
          ZOO_PORT: 2181
          ZOO_SERVERS: server.1=zoo1:2888:3888
        volumes:
        - ./zk-single-kafka-single/zoo1/data:/data
        - ./zk-single-kafka-single/zoo1/datalog:/datalog
~~~

# KAFKA 설정
* zk-single-kafka-single.yml
~~~yaml
version: '3'

services:
    ...

    kafka1:
        image: confluentinc/cp-kafka:5.5.0
        hostname: kafka1
        ports:
        - "9092:9092"
        environment:
          KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka1:19092, LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9092
          KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
          KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
          KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181"
          KAFKA_BROKER_ID: 1
          KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
          KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
        volumes:
        - ./zk-single-kafka-single/kafka1/data:/var/lib/kafka/data
        depends_on:
          - zoo1
~~~
* depends_on: 서비스 의존관계 설정으로 zoo1 서비스에 의존성 설정. zoo1 서비스가 실행되어야 실행됨

# 실행 및 확인
~~~bash
docker-compose -f zk-single-kafka-single.yml up

docker-compose -f zk-single-kafka-single.yml ps
     Name                   Command               State                     Ports                   
----------------------------------------------------------------------------------------------------
kafka_kafka1_1   /etc/confluent/docker/run        Up      0.0.0.0:9092->9092/tcp                    
kafka_zoo1_1     /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tcp
~~~

# 테스트하기
> kafkacat 없으면 설치. brew install kafkacat
* 터미널에서 컨슈머 모드 실행
~~~bash
kafkacat -b localhost:9092 -t new_topic -C 
~~~
* 터미널에서 프로듀서 모드 실행 및 메세지 전송
~~~bash
kafkacat -b localhost:9092 -L -t new_topic -P         
message0001
2
3
4
~~~
* 컨슈머 확인
~~~bash
...

% Reached end of topic new_topic [0] at offset 0
message0001
% Reached end of topic new_topic [0] at offset 1
2
% Reached end of topic new_topic [0] at offset 2
3
% Reached end of topic new_topic [0] at offset 3
4
% Reached end of topic new_topic [0] at offset 4
~~~