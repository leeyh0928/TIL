# Intro
* 레디스 고가용성을 위한 센티넬에 대해 알아본다. 
* 실제 구성해보는 실전편은 추후 추가해보자.
# Features
* 모니터링
* 알림
* Fail over
* 환경 구성 프로바이더
# Sentinel 기반의 클러스터링 기본 구성                                                                                     
                              ┌─────────────┐             ┌─────────────┐  
                              │             │             │             │  
                              │  Slave #1   │             │  Slave #2   │  
                              │             │             │             │  
                              └─────────────┘             └─────────────┘  
                                     ▲                           ▲         
                                     └─────────────┬─────────────┘         
                                                   │                       
                                                   │                       
                                            ┌─────────────┐                
                                            │             │                
                                            │   Master    │                
                                            │             │                
                                            └─────────────┘                
                                                   ▲                       
                                  ┌────────────────┼───────────────┐       
                                  │                │               │       
                                  │                │               │       
                           ┌─────────────┐  ┌─────────────┐ ┌─────────────┐
                           │             │  │             │ │             │
                           │ Sentinel #1 │  │ Sentinel #2 │ │ Sentinel #3 │
                           │             │  │             │ │             │
                           └─────────────┘  └─────────────┘ └─────────────┘
> 기본적으로 3이상의 홀수로 구성해야한다. 바로 ***홀수 단위로 띄운다***
## Sentinel 주요 설정 (sentinel.conf)
~~~
# 포트 번호
port <port number>

# 센티넬이 감시할 Master Redis의 인스턴스 정보
# quorum 이란 의사결정에 필요한 최소 Sentinel 노드수 ***(홀수 단위로 띄우는 전략이 필요한 이유)***
sentinel monitor mymaster <redis master host> <redis master port> <quorum>

# 센티널이 Master 인스턴스에 접속하기 위한 패스워드
sentinel auth-pass mymaster foobared

# 센티널이 Master 인스턴스와 접속이 끊겼다는 것을 알기 위한 최소한의 시간
sentinel down-after-milliseconds mymaster 30000

# 페일오버 작업 시간의 타임오버 시간 설정
# 기본값은 3분
sentinel failover-timeout mymaster 180000

# Master로부터 동기화 할 수 있는 slave의 개수를 지정
# 값이 클수록 Master에 부하가 가중
# 값이 1이라면 Slave는 한대씩 Master와 동기화를 진행
sentinel parallel-syncs mymaster 1
~~~
# References
* https://coding-start.tistory.com/127