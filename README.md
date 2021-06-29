![image](https://user-images.githubusercontent.com/84304043/123741777-95f67700-d8e5-11eb-9c91-089f3049d77f.png)
![image](https://user-images.githubusercontent.com/84304043/123741876-c3432500-d8e5-11eb-9902-5acf7f305446.png)

# 보험가입설계

본 프로젝트는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성하였습니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [보험가입설계](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 프로그래밍 / 폴리글랏 퍼시스턴스](#폴리글랏-프로그래밍-/-폴리글랏-퍼시스턴스)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
    - [Gateway](#Gateway)
    - [CQRS](#CQRS)
    - [Correlation](#Correlation)
  - [운영](#운영)
    - [CI/CD 설정](#CI/CD설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
    - [Self-healing (Liveness Probe)](#Self-healing-(Liveness-Probe))
    - [ConfigMap 사용](#ConfigMap-사용)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
1. 상품관리자가 판매할 보험 상품을 등록/수정한다.
2. 고객이 보험 상품을 선택하여 보험을 가입한다.
3. 보험 가입 등록과 동시에 보험료 결제가 진행된다.
4. 보험료 결제가 완료되면 보험 가입 등록이 완료된다.
5. 보험 가입 등록이 완료되면 심사자가 배정된다.
6. 심사자가 승인하면 보험 계약이 체결된다. 
7. 심사자가 취소하면 보험료 결제와 보험 가입 등록이 취소된다.
8. 고객은 보험 가입에 대한 정보 및 가입 진행 상태를 확인 할 수 있다.


비기능적 요구사항
1. 트랜잭션
    1. 보험료 결제가 되지 않은 보험 가입 등록건은 보험 가입이 성립되지 않는다 - Sync 호출 
1. 장애격리
    1. 보험 심사 기능이 수행되지 않더라도 보험 가입 등록은 365일 24시간 받을 수 있어야 한다 - Async (event-driven), Eventual Consistency
    1. 보험 가입 시스템이 과중되면 사용자를 잠시동안 받지 않고 보험 가입 등록을  잠시후에 하도록 유도한다 - Circuit breaker, fallback
1. 성능
    1. 고객이 보험 가입 진행 상태를 확인할 수 있어야 한다 - CQRS


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
  ![image](https://user-images.githubusercontent.com/24379176/120106054-7a893680-c196-11eb-8a58-479e879d7d80.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/24379176/120106055-7ceb9080-c196-11eb-992e-2913a04eaee5.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://msaez.io/#/storming/nZJ2QhwVc4NlVJPbtTkZ8x9jclF2/every/a77281d704710b0c2e6a823b6e6d973a/-M5AV2z--su_i4BfQfeF


### Event 도출
![image](https://user-images.githubusercontent.com/24379176/120106145-e79ccc00-c196-11eb-8f72-83882d66b57f.png)

### 부적격 Event 탈락
![image](https://user-images.githubusercontent.com/24379176/120106148-e9ff2600-c196-11eb-82d8-2341c88c7774.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - 보험금청구완료됨: 지급처리(보험청구프로세스 최종이벤트)와 중복되며, 다른 팀에서 관심 가질만한 아벤트가 아님
        - 지급안내문발송됨: 다른 팀에서 관심 가질만한 이벤트가 아님
        - 보험가입내역조회됨: 상태(state) 변경을 발생시키지 않음

### Policy 도출
![image](https://user-images.githubusercontent.com/24379176/120106616-c341ef00-c198-11eb-9cb4-c19b454a02de.png)

### Actor, Command 도출
![image](https://user-images.githubusercontent.com/24379176/120106617-c50bb280-c198-11eb-833f-55b4fdab24ec.png)

### Aggregate 도출(View추가)
![image](https://user-images.githubusercontent.com/24379176/120106871-b1ad1700-c199-11eb-907b-4fb552ae4206.png)

    - 청구, 심사, 지급, 청구이력은 그와 연결된 command와 event들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### Bounded Context 도출
![image](https://user-images.githubusercontent.com/24379176/120106873-b2de4400-c199-11eb-80aa-071d0e0d045a.png)

    - 도메인 서열 분리 
        - Core Domain: 심사, 지급 : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 심사의 경우 1주일 1회 미만, 지급의 경우 1개월 1회 미만
        - Supporting Domain: 이력 : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain: 청구 : 고객의 보험금 청구화면으로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![image](https://user-images.githubusercontent.com/24379176/120108615-f1c3c800-c1a0-11eb-9944-428264571b3c.png)

### 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/24379176/120109642-4701d880-c1a5-11eb-8b99-e109ef12930a.png)

    - (ok) 고객이 보험금을 청구한다.
    - (ok) 심사자가 배정된다.
    - (ok) 심사자가 승인을 하면 지급이 접수된다.
    - (ok) 지급완료가 되면 지급처리를 종료한다.

![image](https://user-images.githubusercontent.com/24379176/120109646-49643280-c1a5-11eb-9f65-eef97ddab522.png)
    
    - (ok) 고객은 보험금 청구를 취소할 수 있다.
    - (ok) 보험금 청구를 취소하면 심사를 취소한다.
    - (ok) 심사가 취소되면 보험금 청구 취소가 완료된다.
    
![image](https://user-images.githubusercontent.com/24379176/120109648-4bc68c80-c1a5-11eb-996e-58affd3801c1.png)

    - (ok) 고객은 각 단계별 진행상태를 확인할 수 있다.

### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/24379176/120110652-52ef9980-c1a9-11eb-8e1e-d316560a5fe8.png)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 보험금청구 취소시 심사취소 처리: 심사취소가 선행되어야 보험금청구 취소가 완료되므로 ACID 트렌젝션 적용. 보험금청구 취소시 심사취소 처리에 대해서는 Request-Response 방식 처리
        - 지급 완료시 진행상태변경 처리: 지급에서 청구이력 마이크로서비스로 지급 완료 내용이 전달되는 과정에 있어서 청구이력 마이크로서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
        - 나머지 모든 inter-microservice 트랜잭션: 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.



## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/24379176/120750432-51272c80-c541-11eb-98ca-dff1769bf15a.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이썬으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd claim
mvn spring-boot:run

cd review
mvn spring-boot:run 

cd payment
mvn spring-boot:run  

cd history
python policy-handler.py 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Claim 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

```
package bomtada;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Claim_table")
public class Claim {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Integer customerId;
    private Integer price;
    private String status;
    private Date claimDt;
    
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Integer getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Integer customerId) {
        this.customerId = customerId;
    }
    public Integer getPrice() {
        return price;
    }

    public void setPrice(Integer price) {
        this.price = price;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    public Date getClaimDt() {
        return claimDt;
    }

    public void setClaimDt(Date claimDt) {
        this.claimDt = claimDt;
    }
}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package bomtada;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="claims", path="claims")
public interface ClaimRepository extends PagingAndSortingRepository<Claim, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# claim 서비스의 보험금청구 접수처리
http POST http://localhost:8081/claims customerId=1 price=500 status="Received Claim" claimDt=1622635791863

# review 서비스의 심사승인처리
http PUT http://localhost:8082/reviews/1 claimId=1 examinerId=101 customerId=1 contId=80001 price=450 status="Approved Review" reviewDt=1622635799999

# payment 서비스의 지급완료처리
http PUT http://localhost:8083/payments/1 claimId=1 customerId=1 contId=80001 price=450 status="Completed Payment" paymentDt=1622639541714

# 상태 확인
http http://localhost:8081/claims/1
http http://localhost:8082/reviews/1
http http://localhost:8083/payments/1
```


## 폴리글랏 프로그래밍 / 폴리글랏 퍼시스턴스

이력관리 서비스(history)의 시나리오인 청구상태를 고객이 화면에서 확인 가능하도록 하는 기능의 구현 파트는 해당 팀이 python 을 이용하여 구현하기로 하였다. 해당 파이썬 구현체는 각 이벤트를 수신하여 처리하는 Kafka Consumer와 화면을 제공하는 Flask로 구현되었고, DB는 MySQL를 사용했다.
```
# (history) Kafka Consumer

from kafka import KafkaConsumer
import mysql.connector
import json

# DB 연결 및 생성

consumer = KafkaConsumer('bomtada', bootstrap_servers=['localhost:9092'])

for message in consumer:
    print ("%s:%d:%d: key=%s value=%s" % (message.topic, message.partition,
                                            message.offset, message.key,
                                            message.value))
    # DB 저장                     

```
```
# (history) 청구이력 조회화면

from flask import Flask, request
import mysql.connector

app = Flask(__name__)

@app.route("/history")
def hello():
  # DB 조회 및 화면 생성

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8084)
```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 청구취소(ClaimCanceled)->심사취소(cancelReview) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 심사 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (claim) ReviewService.java

package bomtada.external;

@FeignClient(name="review", url="http://localhost:8082")//, fallback = ReviewServiceFallback.class)
public interface ReviewService {

    @RequestMapping(method= RequestMethod.POST, path="/cancelReview")
    public boolean cancelReview(@RequestBody Review review);

}
```

- 청구취소(cancelClaim) Command를 받은 직후(@PreUpdate) 심사취소를 요청하도록 처리
- 심사취소가 완료되어 true가 반환되면 청구취소됨(claimCanceled) Event를 publish
```
# Claim.java (Entity)

    @PreUpdate
    public void onPreUpdate(){

        bomtada.external.Review review = new bomtada.external.Review();

        review.setClaimId(getId());
        review.setStatus("Canceled Review");

        boolean rslt = ClaimApplication.applicationContext.getBean(bomtada.external.ReviewService.class)
            .cancelReview(review);

        if (rslt) {
            ClaimCanceled claimCanceled = new ClaimCanceled();
            BeanUtils.copyProperties(this, claimCanceled);
            claimCanceled.publishAfterCommit();
        }

    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 심사 시스템이 장애가 나면 청구도 못받는다는 것을 확인:


```
# 심사(review) 서비스를 잠시 내려놓음 (ctrl+c)

# 청구취소 처리
http PUT http://localhost:8081/claims/1 customerId=1 price=500 status="Canceled Claim" claimDt=1622641792891 #Fail
```
![image](https://user-images.githubusercontent.com/24379176/120586594-c5dd6680-c46e-11eb-9cc6-8856f1f7bfd8.png)

```
# 심사(review) 서비스 재기동
cd review
mvn spring-boot:run

# h2 사용으로 심사 수동 생성 필요
http POST http://localhost:8082/reviews claimId=1 examinerId=101 customerId=1 price=500 status="Assigned Examiner" reviewDt=1622635791863

# 청구취소 처리
http PUT http://localhost:8081/claims/1 customerId=1 price=500 status="Canceled Claim" claimDt=1622641792891 #Success
```
![image](https://user-images.githubusercontent.com/24379176/120586585-c1b14900-c46e-11eb-8960-f532e45701b9.png)

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


심사가 승인된 후에 지급 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 지급 시스템의 처리를 위하여 심사접수가 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 심사이력에 기록을 남긴 후에 곧바로 심사승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package bomtada;

@Entity
@Table(name="Review_table")
public class Review {

 ...
    @PostUpdate
    public void onPostUpdate(){
        ReviewApproved reviewApproved = new ReviewApproved();
        BeanUtils.copyProperties(this, reviewApproved);
        reviewApproved.publishAfterCommit();
    }

}
```
- 지급 서비스에서는 심사승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package bomtada;

...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReviewApproved_AssignPayment(@Payload ReviewApproved reviewApproved){

        if(!reviewApproved.validate()) return;

        System.out.println("\n\n##### listener AssignPayment : " + reviewApproved.toJson() + "\n\n");
        
        # 심사승인 처리가 되었으니 지급접수를 한다.
        Payment payment = new Payment();
        payment.setCustomerId(reviewApproved.getCustomerId());
        payment.setContId(reviewApproved.getContId());
        payment.setPrice(reviewApproved.getPrice());
        payment.setStatus("Assigned Payment");
        payment.setPaymentDt(new Date());
        payment.setClaimId(reviewApproved.getClaimId());
        paymentRepository.save(payment);
    }

}
```

지급 시스템은 청구/심사와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 지급 시스템이 유지보수로 인해 잠시 내려간 상태라도 청구/심사 진행에 문제가 없다:
```
# 지급 서비스 (payment) 를 잠시 내려놓음 (ctrl+c)

# 심사승인 처리
http PUT http://localhost:8082/reviews/1 claimId=1 examinerId=101 customerId=1 contId=80001 price=450 status="Approved Review" reviewDt=1622635799999   #Success

# 심사이력 확인
http http://localhost:8082/reviews    # Approved Review 확인
```
![image](https://user-images.githubusercontent.com/24379176/120589427-cb897b00-c473-11eb-8aed-e71816514cca.png)

```
# 지급 서비스 기동
cd payment
mvn spring-boot:run

# 지급이력 확인
http http://localhost:8083/payments     # Assigned Payment 확인
```
![image](https://user-images.githubusercontent.com/24379176/120589436-cf1d0200-c473-11eb-9f4c-ab8d394d651b.png)

## Gateway
Gateway를 통해 마이크로 서비스들의 진입점을 통일하였다.

```
# (gateway) application.yml

server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: claim
          uri: http://localhost:8081
          predicates:
            - Path=/claims/**
        - id: review
          uri: http://localhost:8082
          predicates:
            - Path=/reviews/** 
        - id: payment
          uri: http://localhost:8083
          predicates:
            - Path=/payments/** 
        - id: history
          uri: http://localhost:8084
          predicates:
            - Path=/history/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: claim
          uri: http://claim:8080
          predicates:
            - Path=/claims/**
        - id: review
          uri: http://review:8080
          predicates:
            - Path=/reviews/** 
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/** 
        - id: history
          uri: http://history:8080
          predicates:
            # - Path=/claimHistories/** /progressPages/**
            - Path=/history/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
```
```
# 서비스 호출 테스트

http http://localhost:8088/claims     #Success
http http://localhost:8088/reviews    #Success
http http://localhost:8088/payments   #Success
http http://localhost:8088/history    #Success
```


## CQRS

커맨드 (Create - Insert, Update, Delete : 데이터를 변경) 와 쿼리 (Select - Read : 데이터를 조회)의 책임을 분리한다.
각각의 서비스에서 발생하는 CUD는 서비스 내에서 처리하며, Read는 이벤트 소싱을 통해 history 서비스에서 확인 할 수 있다.

* 고객이 자주 진행상태보기 화면에서 진행상태를 확인할 수 있어야 한다.

전체 이력 조회

![image](https://user-images.githubusercontent.com/24379176/120595209-fc21e280-c47c-11eb-89b3-928829e0ea73.png)

고객별 청구이력 조회

![image](https://user-images.githubusercontent.com/24379176/120595912-127c6e00-c47e-11eb-92d1-3e0a133ae9a5.png)

## Correlation

서비스를 이용해 만들어진 각 이벤트 건은 Correlation-key 연결을 통해 식별이 가능하다.
* Correlation-key로 식별하여 '보험금청구접수됨' 이벤트를 통해 생성된 '심사접수' 건에 대해 '보험금청구취소' 시 동일한 Correlation-key를 가지는 심사 건이 취소되는 모습을 확인한다: (FeignClient 설명부분 참고)

진행상태보기 화면에서 Correlation-key인 claim_id로 조회

![image](https://user-images.githubusercontent.com/24379176/120595929-15775e80-c47e-11eb-8319-86037598f983.png)


# 운영

## CI/CD 설정


* 각 구현체들은 github의 source repository에 구성
* Image repository는 ECR 사용
* yaml파일 기반의 Code Deploy
```
# application deploy

cd insurance-claim/yaml

kubectl apply -f configmap.yaml

kubectl apply -f gateway.yaml
kubectl apply -f claim.yaml
kubectl apply -f review.yaml
kubectl apply -f payment.yaml
kubectl apply -f history.yaml
kubectl apply -f consumer.yaml
```
![image](https://user-images.githubusercontent.com/24379176/120727424-78680480-c515-11eb-860b-93786dbca456.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 청구취소(claim)-->심사취소(review) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 청구취소 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# (claim) application.yml

feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(심사:review) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (review) ReviewController.java

    @RequestMapping(value = "/cancelReview",
                    method = RequestMethod.POST,
                    produces = "application/json;charset=UTF-8")
    public boolean cancelReview(@RequestBody Review review) throws Exception {

        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

- siege 툴에서 PUT 방식은 사용할 수 없어 기존에 feignClient를 호출하는 cancelClaim 이벤트를 POST로 받도록 변경하였다.
```
# (claim) ClaimController.java

@Transactional
@RestController
public class ClaimController {
private static Logger log = LoggerFactory.getLogger(ClaimController.class);

@Autowired
ClaimRepository claimRepository;

@RequestMapping(value = "/claims/{claimId}",
                method = RequestMethod.POST,
                produces = "application/json;charset=UTF-8")
public Claim cancelClaim(@PathVariable Long claimId, @RequestBody Claim claim) throws Exception {
    log.info("### cancelClaim called ###");

        ...
        
    return claim;
}
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
# 청구 생성
http POST http://localhost:8081/claims customerId=6 price=50000 status="Received Claim" claimDt=1622636792891

# 청구 취소 부하테스트
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/claims/1 POST {"customerId":6, "price":50000, "status":"Canceled Claim", "claimDt":1622641792891}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 200     1.22 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.25 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.22 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.28 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.28 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.32 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.54 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.58 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.61 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.70 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.71 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.77 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.76 secs:     104 bytes ==> POST http://localhost:8081/claims/1

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 500     1.82 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.77 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.76 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.76 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.75 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.76 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.76 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.76 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.74 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.75 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.86 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.87 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.69 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.70 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     1.45 secs:     216 bytes ==> POST http://localhost:8081/claims/1

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 200     2.03 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.03 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.07 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.09 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.13 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.19 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.22 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.19 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.30 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     1.85 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.01 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.04 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.05 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.07 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.09 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.17 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     2.22 secs:     104 bytes ==> POST http://localhost:8081/claims/1

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 200     3.64 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     3.75 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     3.77 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     3.80 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     3.77 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     3.77 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     3.77 secs:     216 bytes ==> POST http://localhost:8081/claims/1

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 200     3.79 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     3.82 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     3.85 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     3.88 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     3.99 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     3.99 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.17 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.21 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.23 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.23 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.32 secs:     104 bytes ==> POST http://localhost:8081/claims/1

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨


HTTP/1.1 500     4.18 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     4.12 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.21 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.22 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.24 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.40 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.40 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     4.53 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.28 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.35 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     4.37 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.38 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.24 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.32 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     4.44 secs:     192 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 500     3.91 secs:     216 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.43 secs:     104 bytes ==> POST http://localhost:8081/claims/1
HTTP/1.1 200     4.42 secs:     104 bytes ==> POST http://localhost:8081/claims/1


:

Lifting the server siege...
Transactions:		        1042 hits
Availability:		       65.17 %
Elapsed time:		       59.57 secs
Data transferred:	        0.21 MB
Response time:		        5.48 secs
Transaction rate:	       17.49 trans/sec
Throughput:		        0.00 MB/sec
Concurrency:		       95.89
Successful transactions:        1042
Failed transactions:	         557
Longest transaction:	        8.30
Shortest transaction:	        0.01
 
```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 65.17% 가 성공하였고, 35%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.


## 오토스케일 아웃

앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 심사서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 3프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy review --min=1 --max=10 --cpu-percent=3 -n bomtada
```
- 심사서비스 배포시 yaml에 resource limit 설정을 추가 적용한다:
```
    spec:
      containers:
          ...
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
```

- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://claim:8080/claims/1 POST {"customerId":6, "price":50000, "status":"Canceled Claim", "claimDt":1622641792891}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy review -w -n bomtada
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
review   1/1     1            1           5m
review   1/4     1            1           8m13s
review   1/4     1            1           8m13s
review   1/4     1            1           8m13s
review   1/4     4            1           8m13s
review   1/7     4            1           8m28s
review   1/7     4            1           8m28s
review   1/7     4            1           8m28s
review   1/7     7            1           8m28s
review   1/10    7            1           8m43s
review   1/10    7            1           8m43s
review   1/10    7            1           8m44s
review   1/10    10           1           8m44s
review   2/10    10           2           9m42s
review   3/10    10           3           9m42s
review   4/10    10           4           9m42s
review   5/10    10           5           9m56s
review   6/10    10           6           9m57s
review   7/10    10           7           9m57s
review   8/10    10           8           10m
review   9/10    10           9           10m
review   10/10   10           10          10m
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:                  17626 hits
Availability:                  96.47 %
Elapsed time:                 119.06 secs
Data transferred:               1.86 MB
Response time:                  0.67 secs
Transaction rate:             148.04 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                   99.42
Successful transactions:       17626
Failed transactions:             645
Longest transaction:            4.15
Shortest transaction:           0.00
```


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c1 -v -t300s -r10 --content-type "application/json" 'http://payment:8080/payments'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 200     0.00 secs:     354 bytes ==> GET  /payments
HTTP/1.1 200     0.02 secs:     354 bytes ==> GET  /payments
:

```

- 새버전으로의 배포 시작
```
kubectl apply -f payment_na.yaml  # Readiness Probe 미설정 버전

NAME                           READY   STATUS        RESTARTS   AGE
pod/claim-956c9b89d-m6jg6      1/1     Running       0          31m
pod/gateway-78678646b-fgwms    1/1     Running       0          31m
pod/payment-859c66dbd4-m7pnj   0/1     Terminating   0          5m31s
pod/payment-868dd9c698-wvb22   1/1     Running       0          3s
pod/review-67b6fb4948-qcqrk    1/1     Running       0          31m
...

```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:                   3134 hits
Availability:                  75.37 %
Elapsed time:                  17.65 secs
Data transferred:               1.06 MB
Response time:                  0.01 secs
Transaction rate:             177.56 trans/sec
Throughput:                     0.06 MB/sec
Concurrency:                    0.91
Successful transactions:        3134
Failed transactions:            1024
Longest transaction:            0.52
Shortest transaction:           0.00

```
배포기간중 Availability 가 평소 100%에서 75% 로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
kubectl apply -f payment.yaml  # Readiness Probe 설정 버전

NAME                           READY   STATUS        RESTARTS   AGE
pod/claim-956c9b89d-m6jg6      1/1     Running       0          38m
pod/gateway-78678646b-fgwms    1/1     Running       0          38m
pod/payment-859c66dbd4-csxpm   1/1     Running       0          39s
pod/payment-868dd9c698-wvb22   0/1     Terminating   0          7m12s
pod/review-67b6fb4948-qcqrk    1/1     Running       0          38m
...

```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:                 109148 hits
Availability:                 100.00 %
Elapsed time:                 299.56 secs
Data transferred:              36.85 MB
Response time:                  0.00 secs
Transaction rate:             364.36 trans/sec
Throughput:                     0.12 MB/sec
Concurrency:                    0.96
Successful transactions:      109148
Failed transactions:               0
Longest transaction:            1.05
Shortest transaction:           0.00

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


## Self-healing (Liveness Probe)

- 메모리 과부하를 발생시키는 API를 payment 서비스에 추가하여, 임의로 서비스가 동작하지 않는 상황을 만든다. 그 후 LivenessProbe 설정에 의하여 자동으로 서비스가 재시작되는지 확인한다.
```
# (payment) PaymentController.java

@RestController
public class PaymentController {

    @GetMapping("/callmemleak")
    public void callMemLeak() {
    try {
        this.memLeak();
    } catch (Exception e) {
        e.printStackTrace();
    }
    }

    public void memLeak() throws NoSuchFieldException, ClassNotFoundException, IllegalAccessException {
        Class unsafeClass = Class.forName("sun.misc.Unsafe");
        
        Field f = unsafeClass.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);
        System.out.print("4..3..2..1...");
        try {
            for(;;)
            unsafe.allocateMemory(1024*1024);
        } catch(Error e) {
            System.out.println("Boom!");
            e.printStackTrace();
        }
    }
}

```
- payment 서비스에 Liveness Probe 설정을 추가한 payment_bomb.yaml 생성
```
# Liveness Probe 적용
kubectl apply -f payment_bomb.yaml

# 설정 확인
kubectl get deploy payment -n bomtada -o yaml

...
template:
    metadata:
      creationTimestamp: null
      labels:
        app: payment
    spec:
      containers:
      - image: 879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/user10-payment:bomb
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /actuator/health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 120
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 2
        name: payment
        ports:
        - containerPort: 8080
...

```
- 메모리 과부하 발생
```
kubectl exec -it siege -n bomtada -- /bin/bash

# 메모리 과부하 API 호출
http http://payment:8080/callmemleak

# pod 상태 확인
kubectl get po -w -n bomtada

NAME                       READY   STATUS    RESTARTS   AGE
claim-956c9b89d-m6jg6      1/1     Running   0          127m
gateway-78678646b-fgwms    1/1     Running   0          127m
payment-5b7444449f-mp4kf   1/1     Running   0          9m42s
review-67b6fb4948-qcqrk    1/1     Running   0          127m
siege                      1/1     Running   0          128m
payment-5b7444449f-mp4kf   0/1     OOMKilled   0          10m
payment-5b7444449f-mp4kf   1/1     Running     1          10m
```
- pod 상태 확인을 통해 payment서비스의 RESTARTS 횟수가 증가한 것을 확인할 수 있다.

## ConfigMap 사용

시스템별로 또는 운영중에 동적으로 변경 가능성이 있는 설정들을 ConfigMap을 사용하여 관리합니다.

* configmap.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: bomtada-config
  namespace: bomtada
data:
  api.url.review: http://review:8080
```

* claim.yaml (ConfigMap 사용)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: claim
  namespace: bomtada
  labels:
    app: claim
spec:
  replicas: 1
  selector:
    matchLabels:
      app: claim
  template:
    metadata:
      labels:
        app: claim
    spec:
      containers:
        - name: claim
          image: 879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/user10-claim:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: api.url.review
              valueFrom:
                configMapKeyRef:
                  name: bomtada-config
                  key: api.url.review
```

* kubectl describe pod/claim-7fdc457d8d-sx7mr -n bomtada
```
Name:         claim-7fdc457d8d-sx7mr
Namespace:    bomtada
Priority:     0
Node:         ip-192-168-54-112.ap-southeast-1.compute.internal/192.168.54.112
Start Time:   Fri, 04 Jun 2021 00:09:22 +0000
Labels:       app=claim
              pod-template-hash=7fdc457d8d
Annotations:  kubernetes.io/psp: eks.privileged
Status:       Running
IP:           192.168.44.143
IPs:
  IP:           192.168.44.143
Controlled By:  ReplicaSet/claim-7fdc457d8d
Containers:
  claim:
    Container ID:   docker://3d47bc47a32d9039a555cf56394fb3ad7da6a1e8827b56a7392f639f515a32ce
    Image:          879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/user10-claim:latest
    Image ID:       docker-pullable://879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/user10-claim@sha256:156d492bdb8159ba34b2cd4111896caa9e3441995107b4a0038c0e44811a56fd
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 04 Jun 2021 00:09:23 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:      http-get http://:8080/actuator/health delay=10s timeout=2s period=5s #success=1 #failure=10
    Environment:
      api.url.review:  <set to the key 'api.url.review' of config map 'bomtada-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-pmvj6 (ro)
```
