# Game 이벤트 리워드 기능 구현

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW

# Table of contents

- [예제 - 게임 이벤트 리워드 기능](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)


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


# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/68723566/93055595-2db40d00-f6a6-11ea-92c6-b6e48120c03a.PNG)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과
![image](https://user-images.githubusercontent.com/68723566/93046088-ecb2fd00-f693-11ea-836f-bd166b106df1.png)


### 이벤트 도출
![image](https://user-images.githubusercontent.com/68723566/93055272-9a7ad780-f6a5-11ea-9c04-b09e1d89a335.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/68723566/93055275-9bac0480-f6a5-11ea-8a2e-562c465183b5.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - 미션시작됨, 미션결과전송됨, 리워드가선택됨, 리워드정보가전송됨 :  업무적인 의미의 이벤트가 아니라서 제외

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/68723566/93150528-5e428800-f734-11ea-86ab-cb8d82607c86.PNG)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/68723566/93150530-5f73b500-f734-11ea-80c1-03d49ec30252.PNG)

    - mission과 reward, wallet은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/68723566/93150532-600c4b80-f734-11ea-9a07-1be17bba9755.PNG)

    - 도메인 서열 분리 
        - Core Domain:  mission(front), wallet : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 mission 의 경우 1개월 1회 미만, wallet의 경우 2개월 1회 미만
        - Supporting Domain:   reward : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 70% 이상 uptime 목표, 배포주기는 각 팀의 자율임
        - General Domain:   gift: 상품 교환 서비스로 3rd Party 외부 서비스를 사용하는 것도 경쟁력에 도움이 됨

### 폴리시 부착 

![image](https://user-images.githubusercontent.com/68723566/93150533-60a4e200-f734-11ea-929a-fcd4a940d1c7.PNG)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/68723566/93150534-60a4e200-f734-11ea-8a82-06a6e3b48289.PNG)

   
### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
![image](https://user-images.githubusercontent.com/68723566/93046088-ecb2fd00-f693-11ea-836f-bd166b106df1.png)

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

```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 game-mission 마이크로 서비스).

```
package game;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Mission_table")
public class Mission {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long customerId;
    private Long rewardId;
    private String status;

    @PostPersist
    public void onPostPersist(){
        MissionAchieved missionAchieved = new MissionAchieved();
        BeanUtils.copyProperties(this, missionAchieved);
        missionAchieved.publishAfterCommit();
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Long customerId) {
        this.customerId = customerId;
    }
    public Long getRewardId() {
        return rewardId;
    }

    public void setRewardId(Long rewardId) {
        this.rewardId = rewardId;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package game;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface MissionRepository extends PagingAndSortingRepository<Mission, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# mission 서비스의 미션달성 처리 (POST)
http localhost:8081/missions customerId=11 status=Achieved

# reward 서비스의 조회 (GET)
http localhost:8082/rewards/1

# reward 서비스의 발급 처리 (PATCH)
http localhost:8083/wallets/1 status=Exchanged

# wallet 조회 (GET)
http localhost:8083/wallets/1

```


## 비동기식 호출 


mission 달성이 이루어진 후에 reward에 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 mission 시스템이 블락되지 않아도록 처리한다.
 
- 이를 위하여 mission 달성 기록을 남긴 후에 곧바로 reward를 요청하는 이벤트를 카프카로 송출한다(Publish)
 
```
@Entity
@Table(name="Mission_table")
public class Mission {
...
    @PostPersist
    public void onPostPersist(){
        MissionAchieved missionAchieved = new MissionAchieved();
        BeanUtils.copyProperties(this, missionAchieved);
        missionAchieved.publishAfterCommit();
    }
}
```
- reward 서비스에서는 mission 달성 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
@StreamListener(KafkaProcessor.INPUT)
public void wheneverMissionAchieved_Allocate(@Payload MissionAchieved missionAchieved){

    if(missionAchieved.isMe()){
        Reward reward = new Reward();
        reward.setMissionId(missionAchieved.getId());
        reward.setCustomerId(missionAchieved.getCustomerId());
        reward.setStatus("RewardAllocated");

        rewardRepository.save(reward);
        System.out.println("##### listener Allocate : " + missionAchieved.toJson());
    }
}
```

mission - reward 시스템은 이벤트 수신에 따라 처리되기 때문에, reward 시스템이 유지보수로 인해 잠시 내려간 상태라도 미션을 달성하는데 문제가 없다:
```
# reward 서비스를 잠시 내려놓음 (ctrl+c)

#미션 달성 처리
http localhost:8081/missions costomerId=1 status=Achieved   #Success
http localhost:8081/missions costomerId=2 status=Achieved   #Success

#미션 상태 확인
http localhost:8081/missions/1     # rewardId 안바뀜 확인

#reward 서비스 기동
cd game-reward
mvn spring-boot:run

#주문상태 확인
http localhost:8081/missions/1     # rewardId Update됨을 확인
```

# 평가 항목

## EKS 배포 확인 (kubectl get all -n game)
![image](https://user-images.githubusercontent.com/24929411/93156912-14fa3480-f744-11ea-99ae-efa99fba95ea.png)



## Saga (1)

미션 달성 후 Database에 바로 commit 후 미션을 달성했다는 정보를 reward 서비스에 이벤트를 카프카로 송출한다(Publish)
Mission.java 
```
package game;

@Entity
@Table(name="Mission_table")
public class Mission {
...
    @PostPersist
    public void onPostPersist(){
        MissionAchieved missionAchieved = new MissionAchieved();
        BeanUtils.copyProperties(this, missionAchieved);
        missionAchieved.publishAfterCommit();
    }
...
}
```

데이터 생성 흐름: Mission 달성 --> Reward 지급성 (테스트는 전체 흐름 중 일부분만 캡처했습니다.)
![image](https://user-images.githubusercontent.com/24929411/93293070-113cdf80-f822-11ea-8777-89539c1625a5.png)




## CQRS (2)

Database 조회 업무만을 수행하기 위한 Mypage 개발 

Mypage.java 
```
package game;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Mission_table")
public class Mission {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long customerId;
    private Long rewardId;
    private String status;

    @PostPersist
    public void onPostPersist(){
        MissionAchieved missionAchieved = new MissionAchieved();
        BeanUtils.copyProperties(this, missionAchieved);
        missionAchieved.publishAfterCommit();


    }
```
MissionId와 RewardId를 통해 데이터베이스를 조회한다. 

MypageRepository.java
```
package game;

import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface MypageRepository extends CrudRepository<Mypage, Long> {

    List<Mypage> findByMissionId(Long missionId);
    List<Mypage> findByRewardId(Long rewardId);

}
```
Mission, Reward 에서 Update가 발생하면 mypage에도 적용됨 (mypage MypageViewHandler.java)
```
...
   @StreamListener(KafkaProcessor.INPUT)
    public void whenAllocated_then_UPDATE_1(@Payload Allocated allocated) {
        try {
            if (allocated.isMe()) {
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByMissionId(allocated.getMissionId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setRewardId(allocated.getId());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenIssued_then_UPDATE_2(@Payload Issued issued) {
        try {
            if (issued.isMe()) {
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByRewardId(issued.getId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setRewardStatus(issued.getStatus());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenExchanged_then_UPDATE_3(@Payload Exchanged exchanged) {
        try {
            if (exchanged.isMe()) {
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByRewardId(exchanged.getId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setRewardStatus(exchanged.getStatus());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
...
```


## Correlation (3) 


## 동기식 호출 과 Fallback 처리 (4)

분석단계에서의 조건 중 하나로 wallet -> gift 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- reward 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (wallet) giftService.java

package game.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="gift", url="${api.url.gift}", fallback = GiftServiceFallback.class)
public interface GiftService {

    @RequestMapping(method= RequestMethod.POST, path="/gifts")
    public void exchange(@RequestBody Gift gift);

}
```

- update하기 전(@PreUpdate) gift 서비스에 요청하도록 처리 - (로직상에서 PATCH로 동작하기 때문에 PreUpdate를 사용했습니다.)
```
# Wallet.java (Entity)

    @PreUpdate
    public void onPreUpdate(){
        game.external.Gift gift = new game.external.Gift();
        gift.setWalletId(this.getId());
        gift.setStatus("Exchanged.");
        WalletApplication.applicationContext.getBean(game.external.GiftService.class)
            .exchange(gift);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, gift 시스템이 장애가 나면 위 요청이 실패함:


```
# local test
# gift 서비스를 잠시 내려놓음 (ctrl+c) 

#Exchanged 처리 (PATCH)
http localhost:8083/wallets/1 status=Exchanged  #Fail

#gift 서비스 기동
cd game-gift
mvn spring-boot:run

#Exchanged 처리 (PATCH)
http localhost:8083/wallets/1 status=Exchanged    #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)

## Gateway (5)
* Gateway 프레임워크 선택: Istio

설치된 Istio 확인 
![image](https://user-images.githubusercontent.com/24929411/93156573-563e1480-f743-11ea-855c-c86cbbfb9787.png)

설치된 Istio Gateway yaml 확인
![image](https://user-images.githubusercontent.com/24929411/93156697-9f8e6400-f743-11ea-8a3d-04206639292f.png)

설치된 Istio Virtual Service 확인
![image](https://user-images.githubusercontent.com/24929411/93156795-d2d0f300-f743-11ea-84ff-edf2d868cbab.png)



## CI/CD 설정 (6)


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.

AWS Console Code Build 화면 (여러 Component들 중 일부만 캡처 했습니다.)
![image](https://user-images.githubusercontent.com/24929411/93157202-baada380-f744-11ea-862f-9d60033413c0.png)



## 서킷 브레이킹 (7)

* 서킷 브레이킹 프레임워크의 선택: Istio

시나리오는 wallet-->gift 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Istio DestinationRule 설정: Queue에서 Connection pool 에 연결을 기다리는 request수가 1이 넘어가면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
![image](https://user-images.githubusercontent.com/24929411/93157393-16782c80-f745-11ea-8c75-b92c8e9315a3.png)

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:

(gift 서비스에 GET 요청을 하여 테스트)

동시 사용자 1, 3초 동안 --> 모두 성공
![image](https://user-images.githubusercontent.com/24929411/93169459-19cce180-f760-11ea-9f6e-65b23b9cd5b9.png)

동시 사용자 2, 3초동안 --> 일부 실패 (동시 요청 수가 1이 넘어가면서 CB가 작동되었음을 알 수 있음)
![image](https://user-images.githubusercontent.com/24929411/93169647-77f9c480-f760-11ea-95fd-f24b51751649.png)




- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

## 오토스케일 아웃 (8)
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- mypage 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 10개까지 늘려준다
HPA 설정 확인
![image](https://user-images.githubusercontent.com/24929411/93174730-6c5ecb80-f769-11ea-81c6-a5086cc57e01.png)

현재 game-mypage 의 Pod수 확인
![image](https://user-images.githubusercontent.com/24929411/93174822-a334e180-f769-11ea-937e-3b29597c139a.png)

Siege를 통해 game-mypage Pod에 부하
```
$ siege -c150 -t60S -v http://game-mypage:8080/mypages/1 
```

부하에 따라 game-mypage pod의 CPU 사용율이 증가 했고, Pod Replica 수가 증가 하는것을 확인할 수 있음 
![image](https://user-images.githubusercontent.com/24929411/93176312-0e7fb300-f76c-11ea-86d3-3e56074a4a30.png)




## 무정지 재배포, Readiness probe 설정 (9)

각각 component에 readiness probe 설정 확인 (game-mission pod의 yaml)
![image](https://user-images.githubusercontent.com/24929411/93157685-a5854480-f745-11ea-8e50-884ad7f1f4f8.png)

무정지 재배포 테스트 시나리오
- siege 로 배포작업 직전에 워크로드를 100초동안 모니터링함. 
```
$ siege -c2 -t100S -v http://game-mypage:8080/mypages/1
```
- 모니터링 도중에 game-mypage의 image 버전을 update 
```
kubectl set image deployment/game-mypage game-mypage=271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/game-mypage:f9231b22ed9426a743caa30f64bf6973e171993e
```
- siege의 결과값이 100% 성공이면 무정지 재배포됨을 확인
![image](https://user-images.githubusercontent.com/24929411/93181352-5524db80-f773-11ea-9e50-f9bff8acac2a.png)


## ConfigMap/Persistence Volume (10)

### ConfigMap

Java application.yml 환경변수 적용을 위해 ConfigMap 설정

Mypage Pod (kubectl get deployment game-mypage -o yaml)

configMap으로 부터 변수 가져오는 설정:
![image](https://user-images.githubusercontent.com/24929411/93177206-68cd4380-f76d-11ea-9531-91e63ff91737.png)


configMap에 설정된 데이터 확인 (kubectl get cm teamb-config -o yaml)
![image](https://user-images.githubusercontent.com/24929411/93177339-a03bf000-f76d-11ea-9199-b77ad9867c9d.png)

mypage Application에서 configMap data를 사용하는 부분(mypage의 application.yml)
```
...
  datasource:
    url: jdbc:mariadb://${DB_URL}/teamb-mariadb?useUnicode=yes&characterEncoding=UTF-8
    driver-class-name: org.mariadb.jdbc.Driver
    username: ${DB_USER}
    password: ${DB_PASSWORD}
...
```


### Persistence Volume
데이터를 영구적으로 보관하기 위해 efs 사용.

PVC 확인
![image](https://user-images.githubusercontent.com/24929411/93179564-cb740e80-f770-11ea-9484-430b53e39f5a.png)

game-mypage Deployment에서 해당 pvc를 volumeMount 하여 사용 (kubectl get deployment game-mypage -o yaml)
![image](https://user-images.githubusercontent.com/24929411/93179706-037b5180-f771-11ea-828c-17e59980c0ab.png)



## Polyglot (11)

각 서비스에 맞는 여러가지 데이터베이스 사용 (H2, RDS)

RDS를 사용하는 game-mypage의 application.yml 설정
```
...
  datasource:
    url: jdbc:mariadb://team-mariadb.cuy0kl2qzoel.ap-northeast-2.rds.amazonaws.com:3306/teamb-mariadb?useUnicode=yes&characterEncoding=UTF-8
    driver-class-name: org.mariadb.jdbc.Driver
    username: xxxxx
    password: xxxxxxx
...
```

## Liveness Probe (12)

- Liveness Probe를 통해 특정 경로밑에 파일이 존재하는지 확인 하면서 컨테이너가 동작 중인지 여부를 판단한다. 

Liveness 설정 (테스트를 위해 Persistent Volume 에 미리 healthy 파일을 생성해 놨습니다.)
![image](https://user-images.githubusercontent.com/24929411/93283043-af24b000-f80a-11ea-9867-790b7bababe2.png)

정상적으로 pod실행 (kubectl describe pod game-mypage-xxx-xxx)
![image](https://user-images.githubusercontent.com/24929411/93283808-3c1c3900-f80c-11ea-9624-1e89a9b28807.png)

Liveness 설정 변경 (/mnt/aws/healthy --> /tmp/healthy 로 변경)
```
livenessProbe:
  exec:
    command:
    - cat
    - /mnt/aws/healthy --> /tmp/healthy 로 변경 
  failureThreshold: 5
  initialDelaySeconds: 150
  periodSeconds: 5
  successThreshold: 1
  timeoutSeconds: 2
```

파일이 없으니 Liveness 체크에 실패를 했고, pod를 재기동 시킵니다.
![image](https://user-images.githubusercontent.com/24929411/93284147-ff9d0d00-f80c-11ea-8371-aa8684c661ef.png)

## 개인 Project

![personal](https://user-images.githubusercontent.com/64522956/93314498-cb454300-f844-11ea-8a74-0def3e22474b.png)

## EKS 배포 확인 ($ kubectl get all -n game)

<img width="1447" alt="스크린샷 2020-09-16 오후 5 39 41" src="https://user-images.githubusercontent.com/64522956/93316473-33952400-f847-11ea-9a03-79e0dc54d005.png">

## Saga (1)

미션  달성 후 Database에 바로 commit 후 미션을 달성했다는 정보를 reward 서비스에 이벤트를 송출한다(Publish)

데이터 생성 흐름(1) : Mission 달성 -> Reward 지급 -> Wallet에 생성

<img width="1054" alt="스크린샷 2020-09-16 오후 6 04 06" src="https://user-images.githubusercontent.com/64522956/93316363-0e081a80-f847-11ea-891c-dcfb738f00e6.png">

데이터 생성 흐름(2) : Wallet에서 Reward 교환 -> Gift에 생성(교환 내역) -> Payment에 생성(정산 데이터)

<img width="1019" alt="스크린샷 2020-09-16 오후 6 15 52" src="https://user-images.githubusercontent.com/64522956/93317659-b4a0eb00-f848-11ea-864a-84925c7f07ce.png">

## CQRS (2)

Database 조회 업무만을 수행하기 위한 mypage 개발
<img width="932" alt="스크린샷 2020-09-16 오후 6 17 44" src="https://user-images.githubusercontent.com/64522956/93317884-f16ce200-f848-11ea-889b-d9a26c5e0c50.png">

## ConfigMap, EFS 수정

<img width="773" alt="스크린샷 2020-09-16 오후 6 20 12" src="https://user-images.githubusercontent.com/64522956/93318214-650eef00-f849-11ea-956e-150cb7f3093a.png">

## Gateway, VirtualService, DestinationRule 설정

<img width="590" alt="스크린샷 2020-09-16 오후 6 23 04" src="https://user-images.githubusercontent.com/64522956/93318451-b0290200-f849-11ea-85d9-24fded664e26.png">

kiali 확인

<img width="1766" alt="스크린샷 2020-09-16 오후 6 25 55" src="https://user-images.githubusercontent.com/64522956/93318809-16158980-f84a-11ea-82b1-66bc0348dc5b.png">
