# Intro
> Docker Compose를 이용하여 Redis Sentinel 기반의 클러스터링을 구성해본다.

# 1. Redis Master-Slave(replica) with Docker Compose
> Redis Master와 Slave 2대를 구성하고, 컨테이너 간 Host Name으 통신을 위해 Docker Network을 생성한다.
> 제공되는 환경변수가 더 있지만 간단히 테스트를 위해 Default 설정 사용 및 설정을 최소화하였다.
> 사용한 환경변수에 대한 설명을 아래 추가한다. 추가적인 환경변수는 References를 참고하자.
- ALLOW_EMPTY_PASSWORD : Redis 패스워드 빈값 설정 여부(yes/no)
- REDIS_REPLICATION_MODE : Redis Replication 모드(master/slave)
- REDIS_MASTER_HOST : Redis Slave의 Master 호스트(Host Name or IP)
> docker-compose.yml
~~~yaml
version: '3'

services:
  redis_master:
    image: bitnami/redis:6.0
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
      - REDIS_REPLICATION_MODE=master
    networks:
      - app-tier
    volumes:
      - redis_master_data:/bitnami
  redis_slave1:
    image: bitnami/redis:6.0
    depends_on:
      - redis_master
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis_master
    networks:
      - app-tier
  redis_slave2:
    image: bitnami/redis:6.0
    depends_on:
      - redis_master
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis_master
    networks:
      - app-tier

networks:
  app-tier:
    driver: bridge

volumes:
  redis_master_data:
    driver: local
~~~
# 2. Redis Sentinel with Docker Compose
> 위에서 Redis Master 및 Slave을 3개 구성했으므로 Redis Sentinel도 동잃하게 3개 구성한다.
> docker-compose.yml의 services 항목에 아래 내용을 추가한다.
~~~yaml
  redis-sentinel1:
    image: bitnami/redis-sentinel:6.0
    environment:
      - REDIS_MASTER_HOST=redis_master
    ports:
      - 26379:26379
    networks:
      - app-tier
  redis-sentinel2:
    image: bitnami/redis-sentinel:6.0
    environment:
      - REDIS_MASTER_HOST=redis_master
    ports:
      - 26380:26379
    networks:
      - app-tier
  redis-sentinel3:
    image: bitnami/redis-sentinel:6.0
    environment:
      - REDIS_MASTER_HOST=redis_master
    ports:
      - 26381:26379
    networks:
      - app-tier
~~~
# 3. Docker Compose 실행
~~~bash
docker-compose -f /path/to/docker-compose.yml up -d
~~~
# 4. 실행 결과
## Redis Master
> Master에 2대의 Slave가 연결(***Replica***)된 것을 알 수 있다.
~~~bash
 15:48:15.98 Welcome to the Bitnami redis container
 15:48:15.99 Subscribe to project updates by watching https://github.com/bitnami/bitnami-docker-redis
 15:48:15.99 Submit issues and feature requests at https://github.com/bitnami/bitnami-docker-redis/issues
 15:48:15.99 
 15:48:16.00 INFO  ==> ** Starting Redis setup **
 15:48:16.01 WARN  ==> You set the environment variable ALLOW_EMPTY_PASSWORD=yes. For safety reasons, do not use this flag in a production environment.
 15:48:16.02 INFO  ==> Initializing Redis...
 15:48:16.08 INFO  ==> Configuring replication mode...
 15:48:16.11 INFO  ==> ** Redis setup finished! **

 15:48:16.12 INFO  ==> ** Starting Redis **
1:C 06 May 2020 15:48:16.132 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 06 May 2020 15:48:16.132 # Redis version=6.0.1, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 06 May 2020 15:48:16.132 # Configuration loaded
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 6.0.1 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

1:M 06 May 2020 15:48:16.133 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 06 May 2020 15:48:16.133 # Server initialized
1:M 06 May 2020 15:48:16.133 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 06 May 2020 15:48:16.133 * Ready to accept connections
1:M 06 May 2020 15:48:16.482 * Replica 172.24.0.7:6379 asks for synchronization
1:M 06 May 2020 15:48:16.482 * Full resync requested by replica 172.24.0.7:6379
1:M 06 May 2020 15:48:16.482 * Starting BGSAVE for SYNC with target: disk
1:M 06 May 2020 15:48:16.482 * Background saving started by pid 57
57:C 06 May 2020 15:48:16.484 * DB saved on disk
57:C 06 May 2020 15:48:16.484 * RDB: 0 MB of memory used by copy-on-write
1:M 06 May 2020 15:48:16.499 * Replica 172.24.0.6:6379 asks for synchronization
1:M 06 May 2020 15:48:16.499 * Full resync requested by replica 172.24.0.6:6379
1:M 06 May 2020 15:48:16.499 * Waiting for end of BGSAVE for SYNC
1:M 06 May 2020 15:48:16.535 * Background saving terminated with success
1:M 06 May 2020 15:48:16.536 * Synchronization with replica 172.24.0.7:6379 succeeded
1:M 06 May 2020 15:48:16.536 * Synchronization with replica 172.24.0.6:6379 succeeded
1:M 06 May 2020 15:51:17.910 # Connection with replica 172.24.0.7:6379 lost.
1:M 06 May 2020 15:51:17.910 # Connection with replica 172.24.0.6:6379 lost.
1:M 06 May 2020 15:51:18.742 * Replica 172.24.0.7:6379 asks for synchronization
1:M 06 May 2020 15:51:18.742 * Partial resynchronization request from 172.24.0.7:6379 accepted. Sending 135 bytes of backlog starting from offset 36172.
1:M 06 May 2020 15:51:18.742 * Replica 172.24.0.6:6379 asks for synchronization
1:M 06 May 2020 15:51:18.742 * Partial resynchronization request from 172.24.0.6:6379 accepted. Sending 135 bytes of backlog starting from offset 36172.
~~~
## Redis Slave
> Slave가 Master에 연결(***Replica***)된 것을 알 수 있다.
~~~bash
 15:48:16.39 Welcome to the Bitnami redis container
 15:48:16.39 Subscribe to project updates by watching https://github.com/bitnami/bitnami-docker-redis
 15:48:16.39 Submit issues and feature requests at https://github.com/bitnami/bitnami-docker-redis/issues
 15:48:16.39 
 15:48:16.39 INFO  ==> ** Starting Redis setup **
 15:48:16.40 WARN  ==> You set the environment variable ALLOW_EMPTY_PASSWORD=yes. For safety reasons, do not use this flag in a production environment.
 15:48:16.40 INFO  ==> Initializing Redis...
 15:48:16.45 INFO  ==> Configuring replication mode...
 15:48:16.48 INFO  ==> ** Redis setup finished! **

 15:48:16.49 INFO  ==> ** Starting Redis **
1:C 06 May 2020 15:48:16.496 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 06 May 2020 15:48:16.496 # Redis version=6.0.1, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 06 May 2020 15:48:16.496 # Configuration loaded
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 6.0.1 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

1:S 06 May 2020 15:48:16.496 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:S 06 May 2020 15:48:16.496 # Server initialized
1:S 06 May 2020 15:48:16.496 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:S 06 May 2020 15:48:16.497 * Ready to accept connections
1:S 06 May 2020 15:48:16.497 * Connecting to MASTER redis_master:6379
1:S 06 May 2020 15:48:16.498 * MASTER <-> REPLICA sync started
1:S 06 May 2020 15:48:16.498 * Non blocking connect for SYNC fired the event.
1:S 06 May 2020 15:48:16.498 * Master replied to PING, replication can continue...
1:S 06 May 2020 15:48:16.499 * Partial resynchronization not possible (no cached master)
1:S 06 May 2020 15:48:16.499 * Full resync from master: 9bf5b6d1cfa841c659970101e6bb48d2e161ffe2:0
1:S 06 May 2020 15:48:16.536 * MASTER <-> REPLICA sync: receiving 175 bytes from master to disk
1:S 06 May 2020 15:48:16.537 * MASTER <-> REPLICA sync: Flushing old data
1:S 06 May 2020 15:48:16.538 * MASTER <-> REPLICA sync: Loading DB in memory
1:S 06 May 2020 15:48:16.539 * Loading RDB produced by version 6.0.1
1:S 06 May 2020 15:48:16.539 * RDB age 0 seconds
1:S 06 May 2020 15:48:16.539 * RDB memory usage when created 1.82 Mb
1:S 06 May 2020 15:48:16.539 * MASTER <-> REPLICA sync: Finished with success
1:S 06 May 2020 15:48:16.539 * Background append only file rewriting started by pid 71
1:S 06 May 2020 15:48:16.565 * AOF rewrite child asks to stop sending diffs.
71:C 06 May 2020 15:48:16.565 * Parent agreed to stop sending diffs. Finalizing AOF...
71:C 06 May 2020 15:48:16.565 * Concatenating 0.00 MB of AOF diff received from parent.
71:C 06 May 2020 15:48:16.566 * SYNC append only file rewrite performed
71:C 06 May 2020 15:48:16.566 * AOF rewrite: 0 MB of memory used by copy-on-write
1:S 06 May 2020 15:48:16.599 * Background AOF rewrite terminated with success
1:S 06 May 2020 15:48:16.599 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
1:S 06 May 2020 15:48:16.599 * Background AOF rewrite finished successfully
1:S 06 May 2020 15:51:17.910 # Connection with master lost.
1:S 06 May 2020 15:51:17.910 * Caching the disconnected master state.
1:S 06 May 2020 15:51:17.910 * REPLICAOF 172.24.0.4:6379 enabled (user request from 'id=9 addr=172.24.0.5:41219 fd=12 name=sentinel-ff70109e-cmd age=180 idle=0 flags=x db=0 sub=0 psub=0 multi=4 qbuf=198 qbuf-free=32570 obl=45 oll=0 omem=0 events=r cmd=exec user=default')
1:S 06 May 2020 15:51:17.911 # CONFIG REWRITE executed with success.
1:S 06 May 2020 15:51:18.740 * Connecting to MASTER 172.24.0.4:6379
1:S 06 May 2020 15:51:18.741 * MASTER <-> REPLICA sync started
1:S 06 May 2020 15:51:18.741 * Non blocking connect for SYNC fired the event.
1:S 06 May 2020 15:51:18.741 * Master replied to PING, replication can continue...
1:S 06 May 2020 15:51:18.742 * Trying a partial resynchronization (request 9bf5b6d1cfa841c659970101e6bb48d2e161ffe2:36172).
1:S 06 May 2020 15:51:18.742 * Successful partial resynchronization with master.
1:S 06 May 2020 15:51:18.743 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.
~~~
## Redis Sentinel
> Redis Master를 모니터링 중인 것을 알 수 있다.
~~~bash
redis-sentinel 15:48:15.94 
redis-sentinel 15:48:15.94 Welcome to the Bitnami redis-sentinel container
redis-sentinel 15:48:15.94 Subscribe to project updates by watching https://github.com/bitnami/bitnami-docker-redis-sentinel
redis-sentinel 15:48:15.94 Submit issues and feature requests at https://github.com/bitnami/bitnami-docker-redis-sentinel/issues
redis-sentinel 15:48:15.94 
redis-sentinel 15:48:15.95 INFO  ==> ** Starting Redis sentinel setup **
redis-sentinel 15:48:15.99 INFO  ==> Initializing Redis Sentinel...
redis-sentinel 15:48:15.99 INFO  ==> Configuring Redis Sentinel...
redis-sentinel 15:48:16.05 INFO  ==> ** Redis sentinel setup finished! **

redis-sentinel 15:48:16.06 INFO  ==> ** Starting redis sentinel **
1:X 06 May 2020 15:48:16.068 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:X 06 May 2020 15:48:16.068 # Redis version=6.0.1, bits=64, commit=00000000, modified=0, pid=1, just started
1:X 06 May 2020 15:48:16.068 # Configuration loaded
1:X 06 May 2020 15:48:16.070 * Running mode=sentinel, port=26379.
1:X 06 May 2020 15:48:16.070 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:X 06 May 2020 15:48:16.072 # Sentinel ID is 5f8d75c443283ebfbd0e2c7a6d2deda304d68ae3
1:X 06 May 2020 15:48:16.072 # +monitor master mymaster 172.24.0.4 6379 quorum 2
1:X 06 May 2020 15:48:17.106 * +slave slave 172.24.0.7:6379 172.24.0.7 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 15:48:17.109 * +slave slave 172.24.0.6:6379 172.24.0.6 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 15:48:18.117 * +sentinel sentinel ff70109eb3f15fc6049570b0577534d6917d3634 172.24.0.5 26379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 15:48:18.119 * +sentinel sentinel 37831f9432fafd5728749caf14fcc02a540f7785 172.24.0.3 26379 @ mymaster 172.24.0.4 6379
~~~
# 5. Failover Test
## Redis Master 중지
~~~bash
docker-compose stop <master_node_id>
~~~
## Redis Sentinel 로그 확인
> Master 노드가 중지되자 sdown과 odown이 발생된 것을 알 수 있다.
> Sentinel은 down을 판단하는 방법은 ***sdown***과 ***odown***으로 이루어지고, failover 시 Master 노드가 
> 다운되었다는 판단은 ***다수결의 원칙***을 따른다. (그래서 Sentinel 노드의 개수를 홀수로 구성)
> Master가 다운되면 Sentinel은 자신이 감시하고 있는 Master가 죽었다는 +sdown(***Subjective Down***) 시그널을 발생하는데
> 이는 단지 Sentinel이 Master와 연결되지 않음을 의미한다. 이때 다른 Sentinel도 Master 노드가 중지되었으므로 +sdown 시그널이 발생되고,
> 설정된 quorum 값 이상의 +sdown이 발생되면 +odown(***Objective Down***) 시그널을 발생한다.
> +odown은 다수결에 따라 Master 다운되었다고 명시적으로 선언하는 것과 같고, Sentinel은 Slave 중에 Master를 선출하기 위해 투표를 통해
> Slave 중에 하나를 골라 Master로 승격시키게 된다.
~~~bash
1:X 06 May 2020 16:34:42.005 # +sdown master mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:42.095 # +odown master mymaster 172.24.0.4 6379 #quorum 3/2
1:X 06 May 2020 16:34:42.096 # +new-epoch 1
1:X 06 May 2020 16:34:42.096 # +try-failover master mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:42.099 # +vote-for-leader 5f8d75c443283ebfbd0e2c7a6d2deda304d68ae3 1
1:X 06 May 2020 16:34:42.100 # 37831f9432fafd5728749caf14fcc02a540f7785 voted for 37831f9432fafd5728749caf14fcc02a540f7785 1
1:X 06 May 2020 16:34:42.103 # ff70109eb3f15fc6049570b0577534d6917d3634 voted for 5f8d75c443283ebfbd0e2c7a6d2deda304d68ae3 1
1:X 06 May 2020 16:34:42.155 # +elected-leader master mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:42.155 # +failover-state-select-slave master mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:42.247 # +selected-slave slave 172.24.0.6:6379 172.24.0.6 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:42.247 * +failover-state-send-slaveof-noone slave 172.24.0.6:6379 172.24.0.6 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:42.310 * +failover-state-wait-promotion slave 172.24.0.6:6379 172.24.0.6 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:43.304 # +promoted-slave slave 172.24.0.6:6379 172.24.0.6 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:43.304 # +failover-state-reconf-slaves master mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:43.394 * +slave-reconf-sent slave 172.24.0.7:6379 172.24.0.7 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:44.264 # -odown master mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:44.338 * +slave-reconf-inprog slave 172.24.0.7:6379 172.24.0.7 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:44.338 * +slave-reconf-done slave 172.24.0.7:6379 172.24.0.7 6379 @ mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:44.402 # +failover-end master mymaster 172.24.0.4 6379
1:X 06 May 2020 16:34:44.403 # +switch-master mymaster 172.24.0.4 6379 172.24.0.6 6379
1:X 06 May 2020 16:34:44.403 * +slave slave 172.24.0.7:6379 172.24.0.7 6379 @ mymaster 172.24.0.6 6379
1:X 06 May 2020 16:34:44.403 * +slave slave 172.24.0.4:6379 172.24.0.4 6379 @ mymaster 172.24.0.6 6379
~~~


# References
- https://hub.docker.com/r/bitnami/redis
- https://hub.docker.com/r/bitnami/redis-sentinel