# Kafka: a Distributed Messaging System for Log Processing

## Introduction
현대 시스템에서 로그는 상당히 중요한 역할을 하고있으며 한동안 이벤트를 통한 사용자 참여도 추적, 시스템 사용량, 메트릭 분석을 위해 사용됨.

최근의 트랜드는 이탈률 혹은 세부적인 클릭률(CTR)을 분석하기 위해서도 많이 사용되고 있음. 예를 들어 페이스북의 경우 하루 약 6TB의 다양한 사용자 이벤트 데이터를 수집하고 있음.

다만 초기엔 이러한 데이터를 프로덕션 서버에서 로그 파일을 물리적으로 읽어와 분석하는 방식으로 이루어 졌으며 오래전이지만 페이스북의 [Scribe](https://engineering.fb.com/2019/10/07/core-infra/scribe/) 야후의 `Data Highway` 등이 있었으며 이 시스템들은 주로 로그 데이터를 데이터 웨어하우스나 하둡에 로드하여 오프라인으로 처리하도록 설계됨.
![image](https://github.com/user-attachments/assets/ec22f0a2-0757-4b42-a237-db16641e4f7b)
>
