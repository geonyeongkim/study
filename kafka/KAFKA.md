# KAFKA 정리

## kafka란? 
- 카프카는 메시지 큐를 제공하는 분산 플랫폼.

## kafka 용어 정리.
- zookeeper(주키퍼) : 주키퍼는 코디네이션 분산 플랫폼. kafka와의 연계로는 브로커들의 health check, leader & follower 발탁 등의 일을 함.
- broker(브로커) : 노드를 카프카에서는 브로커라 지칭.
- topid(토픽) : 하나의 토픽은 하나의 논리적인 데이터를 관리하는 큐 그룹.
- partition(파티션) : 한개의 큐로 생각하면 되며, 한 토픽에는 한개 이상의 파티션 존재.
- leader(리더) : 리더는 파티션의 읽기 쓰기를 담당하는 브로커.
- follwer(팔로워) :  팔로워는 리더의 데이터를 복제하는 역할을 담당하는 브로커.
- producer(프로듀서) : 토픽에 메시지를 발행하는 주최. pub을 하는 주최라고 생각하면 됨.
- consumer(컨슈머) : 토픽의 메시지를 소비하는 주최. sub을 하는 주최라고 생각하면 됨.
- consumer-group(컨슈머그룹) : consumer들의 그룹.

## 각 kafka 용어의 상세 설명

1. zookeeper(주키퍼)
    - 주키퍼는 간략하게 카프카 브로커들의 상태 체크 및 메타 데이터들을 znode라는 곳에 저장하여 관리하고 통신하며 관리하는 역할.
    - 주키퍼 클러스터는 앙상블이라는 용어를 사용함.
    - 이전 카프카 토픽의 offset 정보들은 주키퍼에 저장하였음. 하지만 현재는 카프카 브로커에 저장하고, 브로커가 관리하도록 변경되었습니다.
    - 카프카의 리더 선출시에도 주키퍼를 사용합니다. 선출 시 과반수로 선출하기 때문에 앙상블은 홀수로 구성하여야 합니다.

2. broker(브로커)
    - 브로커는 실제로 메시지를 저장하는 카프카 클러스터의 각 노드라고 생각하면 됩니다.
    - 카프카 브로커는 기본적으로 disk 기반이며, 메시지들을 자신의 disk에 write합니다.
        - os cache를 이용하며, cache의 양은 cgroup에 정의한 limit까지 cache가 가능합니다.

3. topic(토픽)
    - 토픽은 카프카에서 사용하는 용어로, 논리적으로 같은 데이터들의 저장소라고 생각하면 됩니다.
    - 토픽은 한개 이상의 파티션이 있어야 합니다.
    - 토픽의 이름은 고유해야 하며, 생성 시 default 파티션 갯수는 8개(confluent 기준), replication 갯수는 3개(confluent 기준) 입니다.
    - 토픽에 대해 다른 설정 값들도 존재합니다.
        - ex) retention.ms -> 토픽의 데이터들을 보관하는 기간 -> [참조](https://kafka.apache.org/23/documentation.html)
    - 각 토픽에는 컨슈머 그룹별로 offsets과 lag 존재
        - offset은 토픽의 데이터가 들어온 순서대로 1씩 증가하게 되는 값.
        - lag는 토픽에서 데이터를 가져가지 못한 값
    
4. partition(파티션) 
    - 파티션은 실제 큐라고 생각하면 됩니다.
    - 각 파티션에는 리더 팔로워가 누구인지 존재 합니다.
    - 파티션 마다 ISR이 존재합니다.
        - ISR은 replication의 group이라고 생각하면 됩니다.
        - 파티션의 리더 팔로워 선출은 이 ISR 그룹내에서 선출하게 됩니다.
    - 실제로 offset과 lag는 파티션마다 존재합니다.
        - topic의 offset은 모든 파티션의 offset의 총합입니다.
        - topic의 lag은 모든 파티션의 lag의 총합입니다.

5. leader(리더)
    - 리더는 데이터의 write/read의 역할을 수행합니다.
    - 리더는 토픽의 replication 갯수만큼 팔로워에게 데이터를 복제하도록 합니다.

6. follower(팔로워)
    - 팔로워는 리더의 요청을 받아 데이터를 복제하는 역할을 합니다.

7. producer(프로듀서)
    - 프로듀서는 메시지를 발행하는 주최입니다.
        - 메시지 pub시 key를 부여할수 있습니다. -> 부여시 키 파티셔닝이 되어 메시지가 적재됩니다.
        - 메시지 본문은 직렬화가 되어 전송됩니다.(String, Json, Avro 직렬화 등이 지원됩니다.)
    - 메시지를 발행하고 브로커에게 ack를 받게됩니다.
        - producer의 ack 설정에 따라 브로커에서 ack 응답을 주는 기준이 달라집니다.
    - 다양한 설정값들이 존재합니다. -> [참조](https://kafka.apache.org/23/documentation.html#producerapi)
    
8. consumer(컨슈머)
    - 메시지를 소비하는 주최입니다.
    - 컨슈머는 commit이라는 행위를 통해 메시지를 소비한것을 브로커에게 알립니다.
        - commit이 정상적으로 작동하게되면 해당 컨슈머 그룹의 lag는 감소하게됩니다.
    - 메시지를 소비하는 행위를 polling(폴링)이라고 합니다.
        - 옵션을 통해 한번의 폴링 시 시간과 메시지 갯수를 지정할수 있습니다.
        - ex) max.poll.records - 한번 폴링의 최대 메시지 갯수
    - 컨슈머의 장애를 판단하기 위해 하트비트 체크와 max.poll.interval.ms 옵션을 통해 poll 유무도 체크를 한다.

9. consumer-group(컨슈머그룹)
    - 컨슈머 그룹은 컨슈머들의 논리적인 그룹이다.
    - 이 컨슈머 그룹을 통해 브로커는 토픽의 offset과 lag를 관리한다.
    - 컨슈머는 컨슈머그룹을 필수로 가져야 합니다. 
    - 컨슈머 그룹에는 토픽의 파티션 갯수만큼 컨슈머들이 존재하는것이 가장 성능이 좋다.
        - 한 파티션은 하나의 컨슈머만 점유할 수 있기 때문.
        > 파티션보다 많은 컨슈머가 존재해도 성능상 이득을 볼 수 없다.
    - 컨슈머 그룹에서 파티션을 점유하고 있던 컨슈머가 down 되었을 시 리밸런싱이라는 행위가 동작한다.
        - 각 컨슈머에게 파티션의 점유를 다시 밸런스 있게 맞춰주는 행위라고 생각하면 됩니다.

## 주키퍼 & 카프카 & 스키마 레지스트리 설치 [install](https://docs.confluent.io/current/installation/installing_cp/zip-tar.html)

- zip/tar로 설치하는 가이드 입니다.
- confluent에서 제공하는 카프카, 주키퍼, 스키마 레지스트리를 설치하여 동작시켰습니다.
    - 스키마 레지스트리는 말그대로 데이터의 스키마의 저장소입니다.
    - 카프카와 연계하여 토픽에 들어가는 메시지의 스키마를 이 스키마 레지스트리에 저장할 수 있습니다.
        - 이는 producer와 consumer가 같은 스키마를 사용하도록 해줍니다.



## cli를 통한 예제

- create topic 
```bash
kafka-topics --create --bootstrap-server dev-geon-kafka001-ncl.nfra.io:9092,dev-geon-kafka002-ncl.nfra.io:9092,dev-geon-kafka003-ncl.nfra.io:9092 --replication-factor 1 --partitions 1 --topic test_geonyeong_topic
```
- list topic
```bash
kafka-topics --list --bootstrap-server dev-geon-kafka001-ncl.nfra.io:9092,dev-geon-kafka002-ncl.nfra.io:9092,dev-geon-kafka003-ncl.nfra.io:9092
```

- send message
```bash
kafka-console-producer --broker-list dev-geon-kafka001-ncl.nfra.io:9092,dev-geon-kafka002-ncl.nfra.io:9092,dev-geon-kafka003-ncl.nfra.io:9092 --topic test_geonyeong_topic
```

- consume message
```bash
kafka-console-consumer --bootstrap-server dev-geon-kafka001-ncl.nfra.io:9092,dev-geon-kafka002-ncl.nfra.io:9092,dev-geon-kafka003-ncl.nfra.io:9092 --topic test_geonyeong_topic --group test_consumer_group --from-beginning
```

- describe topic
```bash
kafka-topics --describe --bootstrap-server dev-geon-kafka001-ncl.nfra.io:9092,dev-geon-kafka002-ncl.nfra.io:9092,dev-geon-kafka003-ncl.nfra.io:9092 --topic test_geonyeong_topic
```

- offset/ lag describe
```bash
kafka-consumer-groups --bootstrap-server dev-geon-kafka001-ncl.nfra.io:9092,dev-geon-kafka002-ncl.nfra.io:9092,dev-geon-kafka003-ncl.nfra.io:9092 --group test_consumer_group --describe
```

## 카프카 매니저 [install](https://m.blog.naver.com/PostView.nhn?blogId=occidere&logNo=221395731049&proxyReferer=https%3A%2F%2Fwww.google.com%2F)

- 카프카 매니저는 yahoo에서 제작한 카프카의 전반적인 관리 및 모니터링을 웹으로 제공해주는 툴입니다.
- 설치 가이드는 위와 link의 블로그대로 따라하면 됩니다. `1.3.3.18 버전은 consumer group 부분에 bug가 있어, 그 윗버전을 설치하시는걸 추천드립니다.`


### axon과 같은 이벤트 framework를 이용하여, 단일이 아닌 여러개의 동일 어플리케이션이 동작하는 환경에서 고려할 점.

1. 
2. 

