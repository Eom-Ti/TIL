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

## 이전 로그 시스템
로그를 전달하기 위한 `Messaging System`중 전통적인 엔터프라이즈 메시징 시스템의 과도한 데이터 처리 방식을 예로 로그 시스템에 적합하지 않음을 이야기함.
1. `IBM Websphere MQ`의 경우 여러 큐에 원자적으로 전달할 수 있는 Transaction을 제공하여 메시지 전달 보장에 중점을 두고있음. 다만 로그 시스템에서는 일부 이벤트에 대한 유실에 관대하기에 이러한 보장이 불필요할 수 있으며 로그 시스템 자체의 복잡성을 증가시킴.

2. `JMS(Java Message Service)`의 경우 여러 메시지를 한 번에 묶어서 보내는(batch)기능이 없어 메시지 마다 개별적인 TCP/IP 왕복 통신이 필요함. 이는 데이터를 대량으로 처리해야하는 로그 시스템에 적합하지 않은 방식임.
   ![image](https://github.com/user-attachments/assets/bad97d21-0d84-4094-803f-59a69a756ee4)
3. 
