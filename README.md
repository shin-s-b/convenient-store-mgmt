![image](https://github.com/shin-s-b/ezdelivery_asis/blob/master/logo_kyobo.png)






# 서점 관리 시스템

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW

# Table of contents

- [서점 관리](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [CQRS](#cqrs)
    - [API 게이트웨이](#api-게이트웨이)
    - [Correlation](#correlation)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트](#비동기식-호출-/-시간적-디커플링-/-장애격리-/-최종-(eventual)-일관성-테스트)
  - [운영](#운영)
    - [CI/CD 설정](#ci/cd-설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-/-서킷-브레이킹-/-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포(Readiness Probe)](#무정지-재배포(Readiness-Probe))
    - [Self-healing(Liveness Probe)](#Self-healing(Liveness-Probe))
    - [Config Map/Persistence Volume](#Config-Map/Persistence-Volume)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항

1. 서점 직원이 서점 본사에 필요 상품을 주문한다.
2. 주문한 내역이 해당 서점 본사에 전달되면 상품을 배송해준다.
3. 배송한 상품을 입고 처리한다.
4. 고객이 상품을 구매한다.
5. 구매 상품에 대한 상품 재고를 변경 처리한다.
6. 서점 직원이 주문한 상품을 취소할 수 있다.
7. 고객이 상품 구매를 취소하면 재고가 변경된다.
8. 서점 직원이 주문 내용, 상품 재고 현황을 조회할 수 있다.
9. 서점 직원이 주문, 배송, 상품 재고, 구매 현황들을 알림으로 받을 수 있다.

비기능적 요구사항

1. 트랜잭션
   1) 배송 취소되지 않으면 주문 취소를 할 수 없어야 한다.  [Sync 호출] 
2. 장애격리
   1) 재고변경 처리가 지연되더라도 고객의 구매는 처리할 수 있도록 유도한다.  [Async (event-driven), Eventual Consistency]
   2) 배송 시스템에 장애가 발생하면 주문취소는 잠시뒤에 처리될 수 있도록 한다. [Citcuit breaker, fallback]
3. 성능
   1. 서점 직원은 주문, 상품 재고 현황에 대해 확인할 수 있어야 한다.  [CQRS]
   2. 서점 직원은 주문, 배송, 상품 재고, 구매 현황들을 알림으로 받을 수 있어야 한다. [Event driven]


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

  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과

* http://www.msaez.io/ 로 모델링한 이벤트스토밍을 하였음

### 이벤트 도출

<img width="871" alt="스크린샷 2021-06-27 오후 5 19 58" src="https://github.com/shin-s-b/ezdelivery_asis/blob/master/1.png">

### 부적격 이벤트 탈락

<img width="870" alt="스크린샷 2021-06-27 오후 5 24 34" src="https://github.com/shin-s-b/ezdelivery_asis/blob/master/2.png">

```
- 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
  - 주문 내용이 서점 본사에 전달됨, 상품 조회됨, 결제버튼 클릭됨, 주문 정보 조회됨, 구매 정보 조회됨 : UI 의 이벤트, 업무적인 의미의 이벤트가 아니라서 제외
```

### 액터, 커맨드 부착하고 어그리게잇으로 묶기

<img width="1223" alt="스크린샷 2021-06-27 오후 6 12 57" src="https://github.com/shin-s-b/ezdelivery_asis/blob/master/3.png">

### 바운디드 컨텍스트로 묶기

<img width="1220" alt="스크린샷 2021-06-29 오후 10 59 22" src="https://user-images.githubusercontent.com/14067833/123810814-aaf6f880-d92d-11eb-836f-dec9f7c6e222.png">

```
- 도메인 서열 분리 
    - Core Domain: order, delivery, product : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 order, delivery 의 경우 1주일 1회 미만, product 의 경우 1개월 1회 미만
    - Supporting Domain: marketing, customer : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
    - General Domain: payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)
```

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

<img width="735" alt="스크린샷 2021-06-30 오전 12 38 36" src="https://user-images.githubusercontent.com/14067833/123827419-902b8080-d93b-11eb-9f4d-7cb49ee8d918.png">

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

<img width="1171" alt="스크린샷 2021-06-30 오전 12 57 48" src="https://user-images.githubusercontent.com/14067833/123830413-38424900-d93e-11eb-82e2-a50f1e04f7b7.png">

### 완성된 1차 모형
<img width="1026" alt="스크린샷 2021-07-05 오전 5 30 15" src="https://user-images.githubusercontent.com/14067833/124398711-1b4cb200-dd52-11eb-9038-f1ee05d972eb.png">


### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

<img width="1026" alt="41E979D4-F6E7-4DBD-956F-2C12BE2EB832" src="https://github.com/shin-s-b/ezdelivery_asis/blob/master/4.png">

```
- 편의점주가 상품을 발주 한다. (OK)
- 본사가 발주된 상품을 배송 한다. (OK)
- 배송이 완료되면 상품을 입고처리 한다. (OK)
- 상품 발주시 편의점주는 view를 통해 발주 상세 내역, 상품 재고현황을 조회할 수 있다. (OK)
```

### 비기능 요구사항에 대한 검증

<img width="1026" alt="9C707C8D-C808-47CB-B4EA-8E94ADA5325C" src="https://github.com/shin-s-b/ezdelivery_asis/blob/master/5.png">

```
- 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
   - 발주 취소시 배송 취소처리: ACID 트랜잭션 적용. 발주 취소시 배송 취소 처리에 대해서는 Request-Response 방식 처리
   - 배송 완료시 상품 입고처리: delivery 에서 product 마이크로서비스로 주문요청이 전달되는 과정에 있어서 product 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
   - 나머지 모든 inter-microservice 트랜잭션: 배달상태, 재고현황 등 모든 이벤트에 대해 카톡을 처리하는 등, 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.
```

### 최종 모델링

<img width="1023" alt="스크린샷 2021-07-08 오후 4 17 17" src="https://user-images.githubusercontent.com/14067833/124879242-fd0fdc00-e007-11eb-8e98-ee63b79a7ed2.png">




## 헥사고날 아키텍처 다이어그램 도출

<img width="1009" alt="스크린샷 2021-07-09 오전 2 55 41" src="https://user-images.githubusercontent.com/14067833/124969014-2c066c00-e061-11eb-90d1-a9f31497843d.png">


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리: 각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```shell
cd gateway
mvn spring-boot:run

cd ../order
mvn spring-boot:run

cd ../delivery
mvn spring-boot:run 

cd ../product
mvn spring-boot:run  

cd ../payment
mvn spring-boot:run  

cd ../alarm
mvn spring-boot:run
```

## CQRS

주문(Order) 내역, 상품(Product) 재고 현황 등 Status에 대하여 서점 직원이 조회 할 수 있도록 CQRS 로 구현하였다.

- OrderStatus

  ```java
  @Entity
  @Table(name="OrderStatus_table")
  public class OrderStatus {
  
      @Id
      @GeneratedValue(strategy=GenerationType.IDENTITY)
      private Long id;
      private Long productId;
      private String status;
      private int quantity;
  }
  ```

- ProductPage

  ```java
  @Entity
  @Table(name="ProductPage_table")
  public class ProductPage {
  
      @Id
      @GeneratedValue(strategy=GenerationType.IDENTITY)
      private Long id;
      private int quantity;
      private int price;
      private String status;
  }
  ```

- ProductPageViewHandler 를 통해 구현

  - "StockModified" 이벤트 발생 시, Pub/Sub 기반으로 별도 ProductPage_table 테이블에 저장

  ```java
  @Service
  public class ProductPageViewHandler {
  
      @StreamListener(KafkaProcessor.INPUT)
      public void whenStockModified_then_UPDATE(@Payload StockModified stockModified) {
        ...
          if (productPageOptional.isPresent()) {
          	ProductPage productPage = productPageOptional.get();
                      
            // view 객체에 이벤트의 eventDirectValue 를 set 함
            productPage.setQuantity(stockModified.getQuantity());
            productPage.setStatus(stockModified.getStatus());
  
            // view 레파지토리에 save
            productPageRepository.save(productPage);
  
          } else {
            // view 레파지토리에 save
            productPageRepository.save(new ProductPage(stockModified.getQuantity(), stockModified.getPrice(), stockModified.getStatus()));
          }
        
        ...
      }
  }
  ```

- OrderStatusViewHandler 를 통해 구현(ProductPageViewHandler 구현 형태 비슷함)

  - "DeliveryStarted", "DeliveryCanceled" 이벤트 발생 시, Pub/Sub 기반으로 별도 OrderStatus_table 테이블에 저장

- View 페이지 조회 결과

  <img width="1029" alt="스크린샷 2021-07-08 오후 3 28 07" src="https://user-images.githubusercontent.com/14067833/124873201-1eb99500-e001-11eb-986e-6484f8c81235.png">

## API 게이트웨이

gateway App을 추가 후 application.yaml에 각 마이크로 서비스의 routes를 추가, 서버의 포트를 8080 으로 설정함

```yaml
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/**, /orderStatuses/**
        - id: delivery
          uri: http://delivery:8080
          predicates:
            - Path=/deliveries/** 
        - id: product
          uri: http://product:8080
          predicates:
            - Path=/products/**, /productPages/**
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/** 
        - id: alarm
          uri: http://alarm:8080
          predicates:
            - Path=/messages/** 
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

```shell
 kubectl apply -f deployment.yml
 kubectl apply -f service.yaml
```


# Correlation

PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 
이벤트 클래스 안의 변수로 전달받아 처리 가능하도록 구현하였다. 

- 주문(Order)시 배송(Delivery), 상품(Product) 등의 상태가 변경되는 걸 확인할 수 있다.
- 주문(Order) 취소를 수행하면 연관된 배송(Delivery), 상품(Product) 등의 상태가 변경되는 걸 확인할 수 있다.



상품 등록

<img width="1158" alt="스크린샷 2021-07-08 오후 2 06 43" src="https://user-images.githubusercontent.com/14067833/124869794-acdf4c80-dffc-11eb-95fd-218e7e5f1d43.png">

주문 (Product 수량 변경 10 -> 13)

<img width="1256" alt="스크린샷 2021-07-08 오후 3 01 35" src="https://user-images.githubusercontent.com/14067833/124872829-b5d21d00-e000-11eb-953e-66af8559c91e.png">

<img width="1120" alt="스크린샷 2021-07-08 오후 3 24 31" src="https://user-images.githubusercontent.com/14067833/124872897-c6829300-e000-11eb-897c-fc19e9130d46.png">

<img width="1009" alt="스크린샷 2021-07-08 오후 3 26 34" src="https://user-images.githubusercontent.com/14067833/124873041-eca83300-e000-11eb-9e69-43c319c63376.png">

주문 취소

- 주문 실행하면 관련 Order, Delivery 데이터는 Delete가 되어 없어지고 관련 Product 수량은 13 -> 10으로 줄어든 것을 볼 수 있으며 OrderStatus, ProductPage에는 이력이 남게 된다. 

<img width="1037" alt="스크린샷 2021-07-08 오후 3 41 48" src="https://user-images.githubusercontent.com/14067833/124874941-4873bb80-e003-11eb-8637-a248c534f150.png">

<img width="1094" alt="스크린샷 2021-07-08 오후 3 42 22" src="https://user-images.githubusercontent.com/14067833/124874997-5c1f2200-e003-11eb-9b2d-13696db80cdc.png">

<img width="1015" alt="스크린샷 2021-07-08 오후 3 42 34" src="https://user-images.githubusercontent.com/14067833/124875047-693c1100-e003-11eb-9c9f-158f87147b94.png">

<img width="1038" alt="스크린샷 2021-07-08 오후 3 43 21" src="https://user-images.githubusercontent.com/14067833/124875081-735e0f80-e003-11eb-9fe0-65b3b8906051.png">


## DDD 의 적용

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Payment 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 이벤트에 의하여 마이크로 서비스들이 상호 작용하기 좋은 모델링으로 구현하였다.

```java
@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long productId;

    private int quantity;
    
    private int price;

    @PostPersist
    public void onPostPersist() {
        PayApproved payApproved = new PayApproved(id, productId, quantity, price);
        payApproved.publishAfterCommit();

    }

    @PreRemove
    public void onPreRemove() {
        PayCanceled payCanceled = new PayCanceled(id, productId, quantity, price);
        payCanceled.publishAfterCommit();
    }

  // getter
...
		
  // setter
...
}
```

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

```java
package convenientstore;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="payments", path="payments")
public interface PaymentRepository extends PagingAndSortingRepository<Payment, Long>{
}
```

- 적용 후 REST API 의 테스트

```shell
# 상품 구매
http POST http://localhost:8088/payments productId=1 quantity=2 price=2000 

# 상품 재고 확인
http GET http://localhost:8088/products/1

# 상품 구매 내역 확인
http GET http://localhost:8088/payments/1
```

## 폴리글랏 퍼시스턴스

Alarm 서비스 특성상 규모가 크지 않고 데이터를 저장하고 빨리 가져올 수 있는 데이터 접근이 빠른 HSQL DB를 사용하기로 하였다. 
h2와 비슷하여 별다른 작업없이 데이터베이스 제품의 설정(pom.xml) 만으로 HSQL DB에 부착시켰다

```xml
<dependency>
  <groupId>org.hsqldb</groupId>
  <artifactId>hsqldb</artifactId>
  <version>2.5.2</version>
  <scope>runtime</scope>
</dependency>
```

- DB 적용 후

<img width="1018" alt="스크린샷 2021-07-09 오전 2 53 28" src="https://user-images.githubusercontent.com/14067833/124968857-fa8da080-e060-11eb-9be4-df166e99b3c1.png">

## 동기식 호출과 Fallback 처리

주문취소(order) -> 배송취소(delivery) 간의 동기식 호출을 하여 일관성을 유지하는 트랜잭션을 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 배송서비스를 호출하기 위하여 Stub과 FeignClient를 이용하여 Service 대행 인터페이스 Proxy를 구현

  ```java
  @FeignClient(name = "delivery", url = "${api.url.delivery}")
  @RequestMapping("/deliveries")
  public interface DeliveryService {
  
      @DeleteMapping(path = "/{deliveryId}")
      public void cancelDelivery(@PathVariable Long deliveryId);
  
  }
  ```

- 주문취소 처리시(@PreRemove) 배송취소를 요청하도록 처리

  ```java
  @Entity
  @Table(name = "Order_table")
  public class Order {
  
      @PreRemove
      public void onPreRemove() {
          Delivery delivery = OrderApplication.applicationContext.getBean(convenientstore.external.DeliveryService.class)
                      .findDelivery(getId());
  
          OrderApplication.applicationContext.getBean(convenientstore.external.DeliveryService.class)
                  .cancelDelivery(delivery.getId());
  
          OrderCanceled orderCanceled = new OrderCanceled(id, productId, quantity, "cancel");
          orderCanceled.publishAfterCommit();
      }
  }
  ```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 배송 시스템이 장애가 발생하면 주문을 받지 못한다.

  ```shell
  # 배송(Delivery)서비스를 잠시 내려놓음
  
  # 주문 취소
  http DELETE http://localhost:8088/orders/3	# Fail
  
  # 배송서비스 재기동
  
  # 주문 취소
  http DELETE http://localhost:8088/orders/3  #Success
  ```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


배송이 이루어진 후에 상품 시스템으로 이를 알려주는 행위는 동기식이 아니라 비동기식으로 처리하였다.

- 배송 내역 처리 후 바로 상품입고 여부 도메인 이벤트를 카프카를 통해 전송한다(Publish)

```java
@Entity
@Table(name = "Delivery_table")
public class Delivery {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long orderId;

    @Column(nullable = false)
    private Long productId;

    private int quantity;

    private String status; // delivery: 배송, cancel: 배송 취소

    @PostPersist
    public void onPostPersist() {
        if ("delivery".equals(this.status)) {
            DeliveryStarted deliveryStarted = new DeliveryStarted(id, orderId, productId, quantity);
            deliveryStarted.publishAfterCommit();
        }
    }
  ...
}
```

- 상품 서비스에서는 입고처리 이벤트를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```java
@Service
public class PolicyHandler{
    private final ProductRepository productRepository;

    public PolicyHandler(final ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverDeliveryStarted_ModifyStock(@Payload DeliveryStarted deliveryStarted){

        if(!deliveryStarted.validate()) {
            return;
        }
        
        Product product = productRepository.findById(deliveryStarted.getProductId()).get();
        product.addStock(deliveryStarted.getQuantity());
        productRepository.save(product);
    }
  ...
}
```

상품시스템이 변경작업으로 다운된 상태라도 배송처리를 하는데 문제가 없게끔 배송시스템과 상품시스템을 완전히 분리하어 이벤트 수신에 따라 처리되게 하였다.

```shell
# 상품(Product)서비스를 잠시 내려놓음

# 주문 요청
http POST http://localhost:8088/orders productId=1 quantity=3 status="order"

# 배송 상태 확인
http GET http://localhost:8088/deliveries?orderId=1		#Success

# 상품(Product)서비스 재기동

# 상품 상태 확인
# 상품시스템 재기동 후 배송 이벤트를 수신하여 상품 입고 처리가 정상완료됨을 확인
http GET localhost:8088/products
```

# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS를 사용하였으며, CodeBuild script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.

- 적용 화면

<img width="1036" alt="스크린샷 2021-07-07 오전 12 29 57" src="https://user-images.githubusercontent.com/14067833/124627470-8c0ede00-deba-11eb-9e1c-49c0cf786d3b.png">

<img width="1048" alt="스크린샷 2021-07-08 오후 12 49 36" src="https://user-images.githubusercontent.com/14067833/124859501-076fad00-dfeb-11eb-9c83-7da7af2330ff.png">

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: istio-injection + DestinationRule

- 시나리오는 (order)-->배송(delivery) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 주문 요청이 과도할 경우 CB 를 통하여 장애격리.

- DestinationRule 를 생성하여 circuit break 가 발생할 수 있도록 설정 최소 connection pool 설정

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: dr-delivery
    namespace: convenientstore
  spec:
    host: delivery
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 1
          maxRequestsPerConnection: 1
  ```

- istio-injection 활성화 및 delivery pod container 확인

  ```shell
  kubectl get ns -L istio-injection
  kubectl label namespace convenientstore istio-injection=enabled 
  ```

  <img width="423" alt="스크린샷 2021-07-08 오후 9 39 09" src="https://user-images.githubusercontent.com/14067833/124922982-f9467e80-e034-11eb-8842-39f127b5b191.png">

  <img width="523" alt="스크린샷 2021-07-08 오후 9 45 08" src="https://user-images.githubusercontent.com/14067833/124923851-ce105f00-e035-11eb-9011-060b1fbea099.png">

- 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:

  - siege 실행

  ```shell
  kubectl run siege --image=apexacme/siege-nginx -n convenientstore
  kubectl exec -it siege -c siege -n convenientstore -- /bin/bash
  ```

  - 동시사용자 1로 부하 생성 시 모두 정상

  ```shell
  siege -c1 -t10S -v --content-type "application/json" 'http://delivery:8080/deliveries POST {"orderId": 1, "productId": 1, "quantity": 3, "status": "delivery"}'
  ```

  <img width="1214" alt="스크린샷 2021-07-08 오후 9 58 06" src="https://user-images.githubusercontent.com/14067833/124925616-999da280-e037-11eb-8b7a-b2c78c8c2121.png">

  - 동시사용자 10로 부하 생성 시 503 에러 발생

  ```shell
  siege -c10 -t10S -v --content-type "application/json" 'http://delivery:8080/deliveries POST {"orderId": 1, "productId": 1, "quantity": 3, "status": "delivery"}'
  ```

  <img width="1224" alt="스크린샷 2021-07-08 오후 10 04 56" src="https://user-images.githubusercontent.com/14067833/124926567-a2db3f00-e038-11eb-9da7-50cb5d04fece.png">

  <img width="577" alt="스크린샷 2021-07-08 오후 10 05 11" src="https://user-images.githubusercontent.com/14067833/124926612-b090c480-e038-11eb-8e79-d1c70fe0673a.png">

  - 다시 최소 Connection pool로 부하시 정상 확인

  <img width="837" alt="스크린샷 2021-07-08 오후 10 20 10" src="https://user-images.githubusercontent.com/14067833/124928638-bbe4ef80-e03a-11eb-8547-190e0bf40412.png">

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 75.24% 가 성공하였고, 25%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.
- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

## 오토스케일 아웃

앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.

- (istio injection 적용한 경우) istio injection 적용 해제

  ```shell
  kubectl label namespace convenientstore istio-injection=disabled --overwrite
  ```

- Delivery deployment.yml 파일에 resources 설정을 추가한다

  <img width="695" alt="스크린샷 2021-07-08 오후 11 07 05" src="https://user-images.githubusercontent.com/14067833/124936706-bb9c2280-e041-11eb-867b-944c52f4b911.png">

- 배송 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다.

  ```shell
  kubectl autoscale deploy delivery -n convenientstore --min=1 --max=10 --cpu-percent=15
  ```

​		<img width="866" alt="스크린샷 2021-07-08 오후 11 21 32" src="https://user-images.githubusercontent.com/14067833/124938476-4a5d6f00-e043-11eb-9624-be9114b74e5d.png">

- CB 에서 했던 방식대로 동시 사용자 50명, 워크로드를 1분 동안 걸어준다.

  ```shell
  siege -c50 -t60S -v --content-type "application/json" 'http://delivery:8080/deliveries POST {"orderId": 1, "productId": 1, "quantity": 3, "status": "delivery"}'
  ```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다.

  ```shell
  kubectl get deploy delivery -w -n convenientstore
  ```

- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다.

  <img width="595" alt="스크린샷 2021-07-08 오후 11 30 17" src="https://user-images.githubusercontent.com/14067833/124940623-1be09380-e045-11eb-9ca8-fca9976dba56.png">

  <img width="593" alt="스크린샷 2021-07-08 오후 11 33 58" src="https://user-images.githubusercontent.com/14067833/124940942-5d713e80-e045-11eb-8d22-1167fa0915ea.png">

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다.

  <img width="684" alt="스크린샷 2021-07-08 오후 11 42 22" src="https://user-images.githubusercontent.com/14067833/124941978-2e0f0180-e046-11eb-8fd0-7ad310212374.png">


## 무정지 재배포(Readiness Probe)

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

#### Readness probe 미설정 상태

- seige 로 배포작업 직전에 워크로드를 모니터링 함.

```shell
siege -c5 -t120S -v --content-type "application/json" 'http://delivery:8080/deliveries POST {"orderId": 1, "productId": 1, "quantity": 3, "status": "delivery"}'
```

- 새버전으로의 배포 시작

```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

  <img width="413" alt="스크린샷 2021-07-09 오전 12 13 51" src="https://user-images.githubusercontent.com/14067833/124947521-dc1caa80-e04a-11eb-9c83-64a7f22f4d86.png">

배포기간중 Availability 가 평소 100%에서 90% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

#### Readness probe 설정 상태

```yaml
# deployment.yml의 readiness probe 설정
# ...
readinessProbe:
  httpGet:
    path: '/actuator/health'
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:

  <img width="462" alt="스크린샷 2021-07-09 오전 12 26 16" src="https://user-images.githubusercontent.com/14067833/124949193-4d109200-e04c-11eb-963a-a4ece38db190.png">

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## Self-healing(Liveness Probe)

- Delivery deployment.yml 파일 수정

  ```
  # ...
  args:
    # /tmp/healthy 파일 생성하고 30초 후 삭제
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
  livenessProbe:
    exec:
      command:
        - cat
        - /tmp/healthy
    initialDelaySeconds: 120
    timeoutSeconds: 2
    periodSeconds: 5
    failureThreshold: 5
  ```

- Delivery 기동후 확인

  ``` shell
  kubectl describe pod delivery -n convenientstore
  ```

kubelet이 5 초마다 livenessProbe를 수행해야 한다고 지정했다(periodSeconds). 하지만 수행하기 전에 120초정도 기다리고(initialDelaySeconds) 수행한다(`cat /tmp/healthy` 명령어 수행). 명령이 성공하면 0을 반환하고 kubelet은 컨테이너가 살아 있고 정상인 것으로 간주합니다. 명령이 0이 아닌 값을 반환하면 kubelet은 컨테이너를 종료하고 다시 시작합니다.

<img width="1117" alt="스크린샷 2021-07-09 오전 1 28 13" src="https://user-images.githubusercontent.com/14067833/124960071-e2188880-e056-11eb-94e1-ef8fcf524a41.png">

<img width="551" alt="스크린샷 2021-07-09 오전 1 28 34" src="https://user-images.githubusercontent.com/14067833/124960112-f2306800-e056-11eb-9b8a-47fce723027a.png">

## Config Map/Persistence Volume

1. configmap.yaml 파일 생성

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: convenientstore-config
   data:
     api.url.delivery: http://delivery:8080
   ```

2. deployment.yml에 적용

   ```yaml
   # ...
             env:
               - name: api.url.delivery	# configmap.yaml에 있는 key-value
                 valueFrom:
                   configMapKeyRef:
                     name: convenientstore-config
                     key: api.url.delivery
   ```

3. 적용 소스

   ```java
   @FeignClient(name = "delivery", url = "${api.url.delivery}")
   @RequestMapping("/deliveries")
   public interface DeliveryService {
   }
   ```

   
