# 실습 - 배달 서비스

# Table of contents
- [실습 - 배달 APP]
  - [서비스 시나리오](#서비스-시나리오)  
  - [분석/설계](#분석설계)
  - [체크포인트](#체크포인트)

# 서비스 시나리오
기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다.
2. 고객은 추가비용을 지불하는 빠른 배송을 선택할 수 있다.
3. 고객이 선택한 메뉴에 대해 결제한다.
4. 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다.
5. 상점주는 주문을 수락하거나 거절할 수 있다.
6. 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다.
7. 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다.
8. 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다.
9. 15분 이내의 거리에 있는 라이더는 추가 요금을 받을 수 있는 빠른 배송건을 선택 가능하다.
10. 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다.
11. 고객이 주문상태를 중간중간 조회한다.
12. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다.
13. 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다.

비기능적 요구사항
1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven


# 분석/설계


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
![image](https://user-images.githubusercontent.com/48579352/206129007-fdd0db5a-7dc3-462c-8c22-96848a2ce91a.png)

    - 고객이 메뉴를 선택하여 주문한다. (ok)
    - 고객은 추가비용을 지불하는 빠른 배송을 선택할 수 있다. (ok)
    - 고객이 선택한 메뉴에 대해 결제한다. (ok)
    - 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다. (ok)
    - 상점주는 주문을 수락하거나 거절할 수 있다. (ok)
    - 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다. (ok)
    - 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다. (ok)
    - 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다. (ok)
    - 15분 이내의 거리에 있는 라이더는 추가 요금을 받을 수 있는 빠른 배송건을 선택 가능하다. (ok)
    - 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다. (ok)
    - 고객이 주문상태를 중간중간 조회한다. (ok)
    - 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다. (ok)
    - 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다. (ok)
    
    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
    - 고객 주문시 결제처리:  주문완료시 결제처리에 대해서는 Request-Response 방식 처리
    - 결제 완료시 점주연결 및 배송처리:  App(front) 에서 Store 마이크로서비스로 주문요청이 전달되는 과정에 있어서 Store 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
    - 나머지 모든 inter-microservice 트랜잭션: 주문상태, 배달상태 등 모든 이벤트에 대해 카톡을 처리하는 등, 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.



## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/48579352/206130434-651d3741-0032-474a-97db-51d4a8fb7eb0.png)

# 체크포인트

- Saga (Pub / Sub)
- CQRS(#CQRS)
- Compensation / Correlation
- Request / Response(# Request / Response)
- Circuit Breaker(# Circuit Breaker)
- Gateway / Ingress(# Gateway / Ingress)

# Saga (Pub / Sub)

Pub. App서비스에서 결제까지 완료된 후에 ordered event 를 Publish.

    @PostPersist
    public void onPostPersist(){
        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        lvpractice.external.PaymentInfo paymentInfo = new lvpracticelast.external.PaymentInfo();
        // mappings goes here
        paymentInfo.setOrderId(getId());

        AppApplication.applicationContext.getBean(lvpractice.external.PaymentService.class).pay(paymentInfo);

        setStatus("결제완료로 주문 접수");

        Ordered ordered = new Ordered(this);
        ordered.publishAfterCommit();
    }

Sub. Store서비스에서 이벤트를 수신

    @Service
    @Transactional
    public class PolicyHandler{
        @Autowired OrderManagementRepository orderManagementRepository;

        @StreamListener(KafkaProcessor.INPUT)
        public void whatever(@Payload String eventString){}

        @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='PaymentApproved'")
        public void wheneverPaymentApproved_OrderReceive(@Payload PaymentApproved paymentApproved){

            PaymentApproved event = paymentApproved;
            System.out.println("\n\n##### listener OrderReceive : " + paymentApproved + "\n\n");




            // Sample Logic //
            OrderManagement.orderReceive(event);




        }

# CQRS
![image](https://user-images.githubusercontent.com/48579352/206613585-cc255131-c9b2-414c-986c-9811dc49e49d.png)

# Compensation / Correlation

# Request / Response
주문과 결제는 동기식으로 구성

주문 Req

    @PostPersist
    public void onPostPersist(){
        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        lvpractice.external.PaymentInfo paymentInfo = new lvpracticelast.external.PaymentInfo();
        // mappings goes here
        paymentInfo.setOrderId(getId());

        AppApplication.applicationContext.getBean(lvpractice.external.PaymentService.class).pay(paymentInfo);

        setStatus("결제완료로 주문 접수");

        Ordered ordered = new Ordered(this);
        ordered.publishAfterCommit();
    }

결제 Res

    @FeignClient(name = "payment", url = "${api.url.pay}", fallback = PaymentInfoServiceImpl.class)
    public interface PaymentInfoService {

        @RequestMapping(method = RequestMethod.POST, path = "/payment")
        public void pay(@RequestBody PaymentInfo paymentInfo);

# Circuit Breaker
서비스를 호출 시 처리 기능에 강제로 딜레이를 준다.

    @PrePersist
    public void onPrePersist(){

    ...    
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 250));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

강제로 부하 발생을 시켜 확인한다.

    - siege -c2 -t10S  -v --content-type "application/json" 'http://localhost:8081/order POST {"menu":"수제햄버거", "count":2, "total_amount":15000}'

# Gateway / Ingress
