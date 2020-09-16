
# Game 이벤트 리워드 기능 구현

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW

# 서비스 시나리오

Game 이벤트

기능적 요구사항
1. 고객이 미션을 달성한다
1. 미션이 달성되면 리워드가 발급이 된다
1. 리워드가 발급이 되면, 지갑에 리워드가 생성이 되고, 미션에 발급된 리워드 ID가 업데이트 된다
1. 고객이 지갑에 있는 리워드를 상품으로 교환한다
1. 리워드가 상품으로 교환이 되면 리워드의 상태가 업데이트 된다
1. 고객이 달성한 미션과 지갑의 상태를 중간중간 조회한다

비기능적 요구사항
1. 트랜잭션
    1. 상품으로 교환이 되지 않으면, 리워드의 상태는 그대로 유지가 된다. Sync 호출 
1. 장애격리
    1. 상품 교환 기능이 수행되지 않더라도 미션 달성 및 리워드 발급은 365일 24시간 진행 될 수 있어야 Async (event-driven), Eventual Consistency
    1. 상품시스템이 과중되면 사용자를 잠시동안 받지 않고 상품 교환을 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 달성한 미션과 지갑의 리워드 상태를 마이페이지(프론트엔드)에서 확인할 수 있어야 한다  CQRS


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과
![image](https://user-images.githubusercontent.com/68723566/93046088-ecb2fd00-f693-11ea-836f-bd166b106df1.png)

* 개인과제로 Notification 추가
![image](https://user-images.githubusercontent.com/31755621/93326389-ad330f00-f853-11ea-92ee-b2d2fca29bd0.png)
1. 미션 달성, 보상 할당, 상품으로 교환 등 이벤트가 발생했을 시에, 이를 통합적으로 발송할 있는 서비스


### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

    - 고객이 미션을 달성한다 (ok)
    - 달성 된 미션 결과가 리워드로 할당 된다 (ok)
    - 할당 된 리워드를 지갑으로 발행한다 (ok)
    - 리워드가 할당 되면 미션 결과에 반영 된다 (ok)    
    - 지갑에 발행된 리워드를 상품으로 교환한다 (ok)
    - 발행 된 결과가 리워드에 반영이 된다 (ok)    
    

### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/68723566/93150535-613d7880-f734-11ea-8bc9-5342aea2cbb0.PNG)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 상품으로 교환이 되지 않으면, 리워드의 상태는 그대로 유지가 된다. Sync 호출
        - 상품 교환 기능이 수행되지 않더라도 미션 달성 및 리워드 발급은 365일 24시간 진행 될 수 있어야 Async (event-driven), Eventual Consistency
        - 고객이 달성한 미션과 지갑의 리워드 상태를 마이페이지(프론트엔드)에서 확인할 수 있어야 한다 CQRS


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/68723566/93185524-7a681880-f778-11ea-8372-7e490d648748.PNG)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd game-mission
mvn spring-boot:run

cd game-reward
mvn spring-boot:run 

cd game-wallet
mvn spring-boot:run  

cd game-gift
mvn spring-boot:run 

cd game-mypage
mvn spring-boot:run 

cd game-notification
mvn spring-boot:run

```

- 적용 후 REST API 
```
# mission 서비스의 미션달성 처리 (POST)
http localhost:8081/missions customerId=11 status=Achieved

# reward 서비스의 조회 (GET)
http localhost:8082/rewards/1

# reward 서비스의 발급 처리 (PATCH)
http localhost:8083/wallets/1 status=Exchanged

# wallet 조회 (GET)
http localhost:8083/wallets/1

# notification 조회 (GET)
http localhost:8086/notification/1

```

## 비동기식 호출

Mission 달성, 보상 할당, 선물 교환 등의 이벤트가 발생시에 알림을 주기 위해서 비 동기식(kafka)로 이벤트를 공유한다.

```

