# Kafka: a Distributed Messaging System for Log Processing

## Introduction
현대 시스템에서 로그는 상당히 중요한 역할을 하고있으며 한동안 이벤트를 통한 사용자 참여도 추적, 시스템 사용량, 메트릭 분석을 위해 사용됨.

최근의 트랜드는 이탈률 혹은 세부적인 클릭률(CTR)을 분석하기 위해서도 많이 사용되고 있음. 예를 들어 페이스북의 경우 하루 약 6TB의 다양한 사용자 이벤트 데이터를 수집하고 있음.

다만 초기엔 이러한 데이터를 프로덕션 서버에서 로그 파일을 물리적으로 읽어와 분석하는 방식으로 이루어 졌으며 오래전이지만 페이스북의 [Scribe](https://engineering.fb.com/2019/10/07/core-infra/scribe/) 야후의 `Data Highway` 등이 있었으며 이 시스템들은 주로 로그 데이터를 데이터 웨어하우스나 하둡에 로드하여 오프라인으로 처리하도록 설계됨.

[scribe](https://blog.det.life/how-did-facebook-design-their-real-time-processing-ecosystem-08da20a61f37)
![image](https://github.com/user-attachments/assets/ec22f0a2-0757-4b42-a237-db16641e4f7b)
> - `Producer`가 로그를 작성해 `Scribe`에 전송함.
> - 로그는 로컬데몬(Scribed) 혹은 Write Service와 같은 원격 서비스에 저장됨.
> - 이후 `LogDevice`에 저장하여 내구성을 확보함.(`HDFS` 를 사용했으나 확장성 한계로 변경)
> - 이를 `Read Service`를 통해 읽어오며 LogDevice의 파티션에서 데이터를 읽고, 이를 대략적인 순서로 병합하여 제공함.

## Related Work
### 전통적인 메시징 시스템
로그를 전달하기 위한 `Messaging System`중 전통적인 엔터프라이즈 메시징 시스템의 특성은 실질적으로 로그 시스템에 적합하지 않음.
1. `IBM Websphere MQ`의 경우 여러 큐에 원자적으로 전달할 수 있는 Transaction을 제공하여 메시지 전달 보장에 중점을 두고있음. 다만 로그 시스템에서는 일부 이벤트에 대한 유실에 관대하기에 이러한 보장이 불필요할 수 있으며 로그 시스템 자체의 복잡성을 증가시킴.

2. `JMS(Java Message Service)`의 경우 여러 메시지를 한 번에 묶어서 보내는(batch)기능이 없어 메시지 마다 개별적인 TCP/IP 왕복 통신이 필요함. 이는 데이터를 대량으로 처리해야하는 로그 시스템에 적합하지 않은 방식임.
   ![image](https://github.com/user-attachments/assets/bad97d21-0d84-4094-803f-59a69a756ee4)

3. 분산 시스템에 대한 지원이 약하다. 여러 서버에 파티셔닝하고 저장하는 쉬운 방법이 존재하지 않으며, 대부분의 메시징 시스템은 메시지가 즉시 소비될 것을 가정하여 대기열(큐)의 사이즈가 작게 설계되어있다. 이는 로그 시스템 처럼 특정한 시점에 많은 양을 가져가야하는 경우 성능저하로 이어진다.

### 현대 로그시스템
해당 논문이 작성된 시기는 2011년으로 해당 시점에 위에서 설명한 Facebook의 Scribe, Yahoo의 Data Highway 등이 HDFS 또는 NFS에 로그를 저장하는 방식으로 특화된 로그시스템을 개발함.
다만 대부분 오프라인 로그 데이터 소비를 위해 설계되어 있고 브로커가 데이터를 consumer에게 전달하는 `PUSH`모델을 사용함.

| 구분 | [LinkedIn의 실시간 데이터 처리 시스템](https://notes.stephenholiday.com/Kafka.pdf)                                    | [Facebook의 실시간 데이터 처리 시스템](https://scontent-gmp1-1.xx.fbcdn.net/v/t39.8562-6/240826476_579340586413244_313948440943029532_n.pdf?_nc_cat=100&ccb=1-7&_nc_sid=e280be&_nc_ohc=rGEi3MCFnpwQ7kNvgFPub8G&_nc_zt=14&_nc_ht=scontent-gmp1-1.xx&_nc_gid=A8zjQJm08srlqGGEBN-PkdT&oh=00_AYCt-e14hAYdxVPiqp2cfO0k4GzNdgULB0KHfjtKmcAZXA&oe=67224EFE)                                        |
|------|-----------------------------------------------------------------|---------------------------------------------------------------------|
| 설계 철학 및 아키텍처 | 분산형 메시징 시스템, 분산 커밋 로그를 사용하여 실시간 및 오프라인 처리 모두 가능                 | 초 단위 지연 목표, 퍼시스턴트 메시지 버스(Scribe)를 통한 데이터 전송 관리, 내결함성 및 확장성 강화       |
| 확장성 및 내결함성 | 브로커 복제 및 자동 리더 선출, 데이터 복구 메커니즘 제공                               | Scribe 메시지 버스를 통한 노드 투명적 결함 처리, 다음 복제 및 클러스터링을 통한 높은 내결함성           |
| 처리 모델 | 최소 한 번(At-least-once) 전달 보장, 데이터 재생 및 과거 데이터 분석 가능              | 여러 전달 보장 옵션 제공(At-most-once, At-least-once 등), 다양한 요구에 맞춘 유연한 처리 모델 |
| 언어 및 API 지원 | Java, Python 등 다양한 언어와 API 지원                                   | SQL 기반의 Puma, Python 기반의 Swift, C++ 기반의 Stylus 등 다양한 언어 지원          |
| 사용 사례 | 로그 집계, 실시간 데이터 스트림, 이벤트 소싱, 마이크로서비스 통신                          | 실시간 보고, 모바일 애플리케이션, 페이지 통찰력, 대시보드 최적화 등 Facebook 내 다양한 용도           |
| **지연 목표**          | 저지연 최적화, 거의 실시간으로 로그 처리 가능                                      | 초 단위의 지연 목표로 설정하여 초저지연보다는 처리량을 우선                                   |
| **처리량**             | 고처리량 지원, 대용량 데이터 효율적으로 처리                                       | 파이프라인 전반에 걸쳐 초당 수백 GB 처리; 대규모 스케일을 위해 설계됨                           |
| **전달 보장**          | 최소 한 번(at-least-once) 전달 보장 제공, 정확히 한 번(exactly-once)은 추가 처리 필요 | 최소 한 번, 최대 한 번(at-most-once) 전달 보장 제공 가능 (플랫폼마다 다름)                 |
| **장애 허용성**        | 복제와 데이터 중복성을 사용하며 Zookeeper에 의존해 조정 수행                          | 영구적 메시지 버스인 Scribe를 통해 내구성과 장애 허용성 제공                               |
| **데이터 처리**        | 주로 로그 집계용이나, 컨슈머 되감기 및 재처리 지원                                   | DAG 구조로 다양한 실시간 처리 기능 지원 (예: 필터링, 집계)                               |

## 카프카 아키텍처
이러한 전통적인 시스템의 한계로 LinkedIn은 메시징 기반의 로그수집기인 Kafka 시스템을 개발하게됨.

![image](https://github.com/user-attachments/assets/d267c6c1-deac-478e-af69-5ffdd027f25b)

Kafka는 **효율적인 데이터 전송**을 위해 메시지를 명시적으로 메모리에 캐시하지 않고 `파일 시스템의 페이지 캐시`만 활용하여 중복 버퍼링을 피하면서 메모리 오버헤드를 감소했음. 이는 Kafka 브로커가 재기동 되어도 캐시가 유지되는 장점이 있음.
> Kafka는 메모리에 cache를 별도로 구현한 것이 아니라 OS의 페이지 캐시를 사용한다. 따라서, OS가 알아서 서버의 남은 memory를 페이지 캐시로 활용하고, 순차 접근이기 때문에 사용될 것으로 예상되는 페이지들을 미리 캐싱해둔다.(read ahead) 또한, Kafka process가 캐시를 관리하는 것이 아니고 OS가 관리하므로, process를 재시작 하더라도 캐시가 그대로 남아있을 수 있다.
> 이 과정에서 Kafka 자체가 디스크에 데이터를 저장하는 명령을 따로 내리기보다는 OS의 캐시 관리 방식에 의존한다. 따라서 만일 카프카를 사용하는 경우 카프카를 실행하는 머신에 설정하는 것을 권장하기도 한다.
> ```text
>주요 설정에는 vm.swappiness(메모리 스왑 빈도 제어), vm.dirty_ratio 및 vm.dirty_background_ratio(페이지 캐시의 데이터 디스크 플러시 빈도 제어) 등이 있다.
>```

카프카가 실제 OS의 메모리 영역(페이지 캐시)을 사용하는 것이 OS 메모리에 영향을 주는 것처럼 보일수 있으나, 실질적으로 페이지 캐시가 가득 차는 것은 리눅스 시스템 입장에서 정상적이며, 오히려 더 효율적인 상황이다.

실질적으로 Linux 디스크 캐시는 매우 작은 영역이며, 예비 메모리를 사용하는 것으로 다른 애플리케이션의 메모리를 더 쓰거나 하는 일은 없으며, 오히려 애플리케이션에서 더 많은 메모리가 필요할 경우 디스크 캐시에서 빌린 청크를 다시 가져오는 방식으로 되어 있다. 리눅스 시스템에선 완전히 사용된 RAM은 경고가 아닌 효율적인 하드웨어 사용으로 보고있다.  
[참조1](https://www.linuxatemyram.com/), [참조2](https://www.linuxatemyram.com/play.html)

또한 `Consumer` 위해 네트워크 접근을 최적화 하며, 이때 리눅스 및 유닉스 운영체제에서 제공하는 `sendfile API`를 활욯하여 세그먼트 파일의 byte를 효율적으로 전달한다.
> 기본적으로 디스크의 데이터를 네트워크를 통해 전달하는 원리는 아래와 같다.
>> ![image](https://github.com/user-attachments/assets/906bb872-201a-4be2-b507-cbb09caa8e38)
>
>위과정을 거치면서 실질적으로 데이터 복사는 총 4번(디스크 -> 읽기 버퍼 -> 사용자 공간 -> 소켓버퍼 -> NIC 버퍼) 발생한다. 리눅스는 이를 sendfile Api를 통해 사용자 공간 개입 없이 복사가 가능하다. 참고로 Java API 중 ` java.nio.channels.FileChannel의 transferTo()` 메서드 또한 애플리케이션의 간섭 없이 데이터를 복사하는 기능을 제공한다. 
>> ![image](https://github.com/user-attachments/assets/054e0715-633a-43d8-aa44-c90174e8db40)
>
> 이를 사용해 카프카는 [zero-copy](https://kafka.apache.org/documentation/#maximizingefficiency) 기법을 활용하여 consumer의 읽기 요청에 대한 처리 속도를 향상시킨다. 이 기법은 consumer가 데이터를 요청할 때마다 데이터를 사용자 공간(user space)에 복사하지 않고 읽기 버퍼에 데이터를 저장하여 전달하는 방식으로 이루어 진다. [참조](https://developer.ibm.com/articles/j-zerocopy/)
> 
> RabbitMQ의 경우 인덱스 기반 스토리지 시스템을 사용하며, 데이터는 일반적으로 인덱스에 액세스 하여 데이터의 물리적인 위치를 찾은 후 저장 매체에서 메모리로 데이터를 읽어 복사하는 방식이다. 이러한 방식에서 개별 메시지에 대한 빠른 액세스 제공을 위해 데이터를 트리 구조에 보관하며 이러한 트리 구조가 제공하는 빠른 개별 읽기는 쓰기 오버헤드의 비용으로 발생하여 로그 방식인 Kafka에 비해 쓰기 처리량 감소, 쓰기 지연 시간 증가 등으로 나타날 수 있음.

위의 내용 이외에도 실질적으로 많은 효율적인 전송 방식이 있으나 이는 우리가 알고있는 내용과 비슷하여 생략하였다. 결론적으로 Kafka 설계중 중요한 부분은 결국 consumer가 이전 오프셋으로 되돌아가 데이터를 다시 읽을 수 있다는 점이며, 이는 ETL(Event Tracing Log) 데이터 로드와 같은 경우 중요한 내용이다.

## 분산환경
Kafka는 메시지 소비시 컨슈머 그룹 개념을 사용하며, 이때 각 컨슈머 그룹은 topic내 파티션을 그룹 내의 컨슈머들 끼리 나눠서 처리하며 하나의 컨슈머에만 전달 된다.
또한 topic 내의 파티션을 병렬 처리의 최소 단위로 설정하여 같은 파티션은 그룹 내에서 한 번에 하나의 컨슈머만 소비하게 한다. 이를 통해 여러 컨슈머 간의 충돌 및 복잡한 조정 과정을 줄이며 조정 과정은 `rebalancing`에서만 발생한다


또한 Zookeeper를 통해 Kafka 브로커와 컨슈머 상태를 관리하며 재조정 이벤트를 감지한다. 다만 `Apache Kafka 4.0`에선 주키퍼를 제거할 계획이며 이는 내부적으로 관리되는 메타데이터용 프토로콜인 `카프카 라프트(Kafka Raft)` 또는 `크라프트(KRaft)`로대체될 예정이다.

Kafka에서 새로운 컨슈머가 시작되거나, 브로커/컨슈머 변경 사항이 생기면 컨슈머는 자신이 처리할 파티션을 다시 할당받기 위해 `리밸런스(rebalance)`라는 과정을 거친다. 이때 `Zookeeper`에서 각 주제에 구독된 파티션과 컨슈머 목록을 가져와, 각 컨슈머가 고유하게 처리할 파티션을 나누고 소유권을 설정한다. 설정이 완료되면 컨슈머는 자신이 맡은 파티션에서 데이터를 가져오는 작업을 시작한다.
컨슈머가 여러 명 있을 경우, 이 알림이 컨슈머마다 약간씩 시간차로 전달될 수 있다. 이로 인해 한 컨슈머가 다른 컨슈머가 소유한 파티션을 차지하려는 경우가 생길 수 있는데, 이때는 먼저 차지하려던 컨슈머가 모든 소유 파티션을 잠시 포기하고 기다린 후 다시 시도하여, 결국 각 파티션의 소유권이 안정적으로 배분된다.

```text
Algorithm 1: rebalance process for consumer Ci in group G
For each topic T that Ci subscribes to {
 remove partitions owned by Ci from the ownership registry
 read the broker and the consumer registries from Zookeeper
 compute PT = partitions available in all brokers under topic T
 compute CT = all consumers in G that subscribe to topic T
 sort PT and CT
 let j be the index position of Ci in CT and let N = |PT|/|CT|
 assign partitions from j*N to (j+1)*N - 1 in PT to consumer Ci
 for each assigned partition p {
 set the owner of p to Ci in the ownership registry
 let Op = the offset of partition p stored in the offset registry
 invoke a thread to pull data in partition p from offset Op
 }
}
```
### 알고리즘 설명
1. **현재 할당된 파티션 해제**: 소비자 \( Ci \)는 자신에게 할당된 파티션을 소유권 레지스트리에서 제거해 초기화한다.
2. **레지스트리 읽기**: Zookeeper에서 브로커와 소비자 레지스트리를 읽어온다.
3. **파티션 및 소비자 계산**:
   - \( PT \): 현재 주제 \( T \)의 모든 브로커에 있는 파티션 집합을 계산한다.
   - \( CT \): 소비자 그룹 \( G \)에서 주제 \( T \)에 구독된 모든 소비자를 계산한다.
4. **정렬 및 인덱스 계산**:
   - 파티션 집합 \( P_T \)와 소비자 집합 \( C_T \)를 정렬한다.
   - 소비자 \( Ci \)의 인덱스 \( j \)를 구하고, \( N \)을 \( PT \)의 파티션 수를 \( CT \)의 소비자 수로 나눈 값으로 설정한다.
5. **파티션 할당**:
   - \( j \times N \)부터 \( (j+1) \times N - 1 \)까지의 파티션을 소비자 \( Ci \)에게 할당한다.
6. **데이터 가져오기**:
   - 할당된 각 파티션 \( p \)에 대해, 소비자 \( Ci \)를 해당 파티션의 소유자로 설정한다.
   - 오프셋 레지스트리에 저장된 파티션 \( p \)의 오프셋 \( Op \)에서부터 데이터를 가져오는 스레드를 시작한다.

## 전달보장
Kafka는 최소 한 번 전달(at-least-once)를 보장하며, 특정 상황에서는 중복 메시지가 발생할 수 있어 애플리케이션 수준에서 중복제거가 필요하다. 이때 말하는 특정상황으론 컨슈머가 메시지 처리 후 오프셋을 기록하지 못하고 비정상적으로 종료될 경우 다른 컨슈머가 해당 파티션에서 중복된 메시지를 읽는 상황이 발생할 수 있다.

해당 논문에선 브로커 장애 시 소비되지 않은 메시지가 손실될 수 있기에 **향후 복제 기능**을 추가할 계획임을 이야기 하는데 해당 기능은 현재 제공을 하고있으며, `Replication Factor` 가 해당 내용이다.

## LinkedIn의 Kafka
![image](https://github.com/user-attachments/assets/dd6a124e-1a8d-4aa2-80ad-bad4cc67df45)

카프카는 각 데이터 센터에 클러스터가 배포되어 있으며, 프론트엔드 서비스에서 생성된 로그 데이터를 로컬 Kafka에 전송한다.
이를 하드웨어 로드벨런서를 사용해 Kafka 브로커 집합에 분배하여 전달하고 온라인 처리는 각 데이터 센터 내에서 이루어지며, 오프라인 분석을 위한 처리는
Hadoop 클러스터 가까이에 위치하여 데이터를 가져오는 방식이다.

해당 클러스터에선 데이터 로드, 리포팅, 분석등의 작업을 수행한다. 또한 데이터 손실 방지를 위해 메시지 타임스탬프와 서버 이름을 기록 하고, 프로듀서가 정해진 시간 내에 게시한 메시지의 양을 파악할 수 있는 모니터링 이벤트를 통해 컨슈머가 수신한 메시지 수와 프로듀서가 기록한 메시지의 수를 비교하여 데이터 손실을 방지한다.

또한 MapReduce 작업으로 데이터를 HDFS에 저장하며 [Avro](https://blog.techeer.net/%EC%B9%B4%ED%94%84%EC%B9%B4-%EB%A9%94%EC%8B%9C%EC%A7%80%EC%97%90-%EC%8A%A4%ED%82%A4%EB%A7%88%EB%A5%BC-%EC%A0%95%EC%9D%98%ED%95%B4-%EB%B3%B4%EC%9E%90-apache-avro-7162e250ae69) 직렬화 프로토콜을 사용해 스키마의 변화를 관리하고 데이터 호환성을 유지한다.
>카프카 프로듀서의 필수 구성에는 직렬처리기가 포함된다. String, Integer, ByteArray 직렬처리기가 있지만 이것들로 모든 데이터의 직렬화를 충족시킬 수 없으므로 다른 직렬 처리기가 필요하다. 그래서 커스텀 직렬처리기를 사용할 수 있지만, 타입의 변화가 있거나 필드가 추가 혹은 삭제된다면 기존 메시지와 새로운 메시지 사이에 호환에 문제가 생긴다.
>
>이런 이유로 JSON, Apache Avro, Thrift, Protobuf 같은 범용 직렬처리기와 역직렬처리기 사용을 권하고 있다

## 성능 테스트

이 실험에서는 Apache Kafka, Apache ActiveMQ, 그리고 RabbitMQ의 성능을 다양한 조건에서 비교하여 측정함. 각 시스템의 메시지 처리 성능과 효율성을 측정하기 위해 유사한 설정을 적용했으며, 총 천만 개의 메시지를 전송하면서 처리량을 기록함.
### 실험 환경
- 하드웨어 설정: 2GHz 8코어 CPU, 16GB 메모리, RAID 10 구성을 가진 6개의 디스크로 구성된 2대의 리눅스 서버를 사용.
- 네트워크 구성: 두 서버는 1Gb 네트워크 링크로 연결.
- 배치 크기: Kafka는 배치 사이즈를 1개와 50개로 설정했지만, ActiveMQ와 RabbitMQ는 배치 전송 기능이 부족하여 메시지를 하나씩 전송.
### 실험 조건
- 메시지 크기 : 200바이트
- 프로듀서 테스트 : 비동기 전송을 통해 각 시스템의 브로커가 메시지를 최대한 빠르게 저장하도록 설정
### 결과 요약
#### 성능
- 처리량: Kafka는 배치 크기가 1일 때 초당 약 5만 개의 메시지를, 배치 크기가 50일 때 초당 약 40만 개의 메시지를 처리. 이는 ActiveMQ와 RabbitMQ보다 훨씬 높은 성능으로, 특히 RabbitMQ보다 약 2배 이상 높았음.


- 이유: Kafka의 비동기 전송 방식은 브로커의 응답을 기다리지 않고 메시지를 전송함으로써 처리량을 극대화함. 특히 배치 전송 시 프로듀서와 브로커 간 1Gb 네트워크 링크를 거의 포화시킬 정도로 데이터를 빠르게 전송함.

  ![image](https://github.com/user-attachments/assets/f896f745-941e-49f2-8ed1-d79d508473af)

#### 저장 포맷의 효율성
- 메시지 오버헤드: Kafka의 메시지 오버헤드는 평균 9바이트였으나, ActiveMQ는 144바이트로 훨씬 더 많았음. 이는 ActiveMQ가 Kafka보다 70% 더 많은 저장 공간을 사용하게 했으며, 이는 JMS의 무거운 메시지 헤더와 인덱싱 구조로 인한 것.


- 추가 효율성: ActiveMQ는 메시지 메타데이터와 상태 관리를 위해 B-Tree 구조에 접근하며 CPU 자원을 소모하는 반면, Kafka는 효율적인 메시지 저장 방식을 통해 이러한 오버헤드를 줄임.

#### 배치 전송의 효율성
- Kafka에서 배치 크기를 50으로 설정한 경우, RPC 오버헤드를 줄이고 처리율을 극대화하여 거의 10배 가까운 성능 향상을 이룸.

### 소비자 성능
- 처리량: Kafka는 초당 22,000개의 메시지를 소비할 수 있었으며, 이는 ActiveMQ와 RabbitMQ보다 4배 이상 높은 수준.


- 이유: Kafka는 더 효율적인 저장 포맷을 사용해 브로커에서 컨슈머로 전송되는 바이트 수를 줄였으며, ActiveMQ와 RabbitMQ의 브로커는 각 메시지의 전달 상태를 유지하느라 추가적인 자원을 소모.


- 기술적 최적화: Kafka는 sendfile API를 통해 전송 오버헤드를 줄여 효율성을 더욱 높임.
  ![image](https://github.com/user-attachments/assets/3a10635e-e93b-4506-8b9c-ab74e7638266)

## 다른 비교 테스트
### Kafka vs Rabbit MQ
- 초당 약 15,000개 메시지 이전
  - RabbitMQ로 메시지를 보내고 받는데 걸리는 대기 시간이 Kafka에 비해 절반의 수준임.
  - 일반적으로 Kafka의 Producer, Consumer의 CPU 사용량이 RabbitMQ에 비해 2배가량 높음
  - Kafka의 경우 디스크에 적극적으로 쓰고 있는 반면 RabbitMQ는 거의 건드리지 않음.
  - 다만 RabbitMQ의 CPU사용량이 50%에 도달하면 대기시간이 증가하기 시작함.
  ![image](https://github.com/user-attachments/assets/5780848f-b051-42aa-85b6-dd764369ba3f)
- 초당 약 15,000개의 메시지
  - RabbitMQ의 CPU 사용량은 100%에 가깝고 성능도 저하되기 시작하며 지연시간이 늘어나기 시작함.
    ![image](https://github.com/user-attachments/assets/45331a87-1f08-4b61-81b1-ffe8415ef97a)
- 초당 약 30,000개의 메시지
  - RabbitMQ의 Producer와 Consumer는 기본 5초 제한 시간에 도달하기 시작하며 실패하기 시작함.
  - Kafka의 경우 오래전에 CPU 사용량이 100%에 도달했지만 Latency가 증가하는 것 외 여전히 모든 메시지를 처리할 수 있음.
  ![image](https://github.com/user-attachments/assets/cb17f28f-4613-46d3-bfe5-5cb817a4e1a1)

Kafka가 Rabbit MQ보다 처리량이 훨씬 높다는 것을 알 수 있음. 다만 기본적으로 Rabbit MQ의 Latency가 훨씬 더 낮음을 알 수 있으며, 이 이유가 Kafka보다 RabbitMQ를 선택하는 주된 이유일 수 있음(유지 관리가 더 간단하고 더 나은 대기시간 성능을 제공함.)

낮은 대기 시간이 필요한 경우 Rabbit MQ를 선택할 수 있는 지표임. 다만 지속적인 측면 혹은 이후의 처리 과정에서 봤을때 Kafka가 좀 더 안정적인 측면이 드러남.[참조](https://www.youtube.com/watch?v=UPkOsXKG4ns)
