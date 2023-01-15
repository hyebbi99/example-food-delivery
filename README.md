![image](https://user-images.githubusercontent.com/487999/79708354-29074a80-82fa-11ea-80df-0db3962fb453.png)

# 서비스 시나리오

배달의 민족 커버하기 - https://1sung.tistory.com/106

기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다
1. 고객이 결제한다
1. 주문이 되면 주문 내역이 입점상점주인에게 전달된다
1. 상점주인이 확인하여 요리해서 배달 출발한다
1. 고객이 주문을 취소할 수 있다
1. 주문이 취소되면 배달이 취소된다
1. 고객이 주문상태를 중간중간 조회한다
1. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다


![image](https://user-images.githubusercontent.com/85150301/212534625-2b76855f-d871-4627-93da-dfdccc0c5f41.png)

## DDD 의 적용

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 JAVA로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd Customer
mvn spring-boot:run

cd Front
mvn spring-boot:run 

cd Rider
mvn spring-boot:run  

cd Store
mvn spring-boot:run 

```

### 1.	포트확인하기

```
gitpod /workspace/food-delivery (main) $ http GET localhost:8081/
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Sun, 15 Jan 2023 08:14:51 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "orders": {
            "href": "http://localhost:8081/orders{?page,size,sort}",
            "templated": true
        },
        "profile": {
            "href": "http://localhost:8081/profile"
        }
    }
}
```

### 2. 주문하기
```
gitpod /workspace/food-delivery (main) $ http POST localhost:8081/orders orderId=a menuInfo=ramen qty=1
HTTP/1.1 201 
Connection: keep-alive
Content-Type: application/json
Date: Sun, 15 Jan 2023 08:15:09 GMT
Keep-Alive: timeout=60
Location: http://localhost:8081/orders/1
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "address": null,
    "customerId": null,
    "deliveryStatus": null,
    "menuInfo": "ramen",
    "orderId": "a",
    "qty": 1,
    "storeId": null
}

gitpod /workspace/food-delivery (main) $ http POST localhost:8081/orders orderId=b menuInfo=rice qty=2
HTTP/1.1 201 
Connection: keep-alive
Content-Type: application/json
Date: Sun, 15 Jan 2023 08:15:20 GMT
Keep-Alive: timeout=60
Location: http://localhost:8081/orders/2
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/2"
        },
        "self": {
            "href": "http://localhost:8081/orders/2"
        }
    },
    "address": null,
    "customerId": null,
    "deliveryStatus": null,
    "menuInfo": "rice",
    "orderId": "b",
    "qty": 2,
    "storeId": null
}
```

### 3.	주문확인하기
```
gitpod /workspace/food-delivery (main) $ http GET localhost:8081/orders/1
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Sun, 15 Jan 2023 08:15:26 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "address": null,
    "customerId": null,
    "deliveryStatus": null,
    "menuInfo": "ramen",
    "orderId": "a",
    "qty": 1,
    "storeId": null
}


gitpod /workspace/food-delivery (main) $ http GET localhost:8081/orders/2
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Sun, 15 Jan 2023 08:15:28 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/2"
        },
        "self": {
            "href": "http://localhost:8081/orders/2"
        }
    },
    "address": null,
    "customerId": null,
    "deliveryStatus": null,
    "menuInfo": "rice",
    "orderId": "b",
    "qty": 2,
    "storeId": null
}
```

### 4.	주문상태 수정하기 
```
gitpod /workspace/food-delivery (main) $ http PATCH http://localhost:8081/orders/1 deliveryStatus=notStarted
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/json
Date: Sun, 15 Jan 2023 08:27:32 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "address": null,
    "customerId": null,
    "deliveryStatus": "notStarted",
    "menuInfo": "ramen",
    "orderId": "a",
    "qty": 1,
    "storeId": null
}
```

## Saga (Pub/Sub)
```
    public static void orderInfoTransfer(Ordered ordered){
        StoreOrder storeOrder = new StoreOrder();

        storeOrder.setOrderId(ordered.getOrderId());
        storeOrder.setCustomerId(ordered.getCustomerId());
        storeOrder.setDeliveryStatus(ordered.getDeliveryStatus());
        storeOrder.setAddress(ordered.getAddress());
        storeOrder.setQty(ordered.getQty());

        repository().save(storeOrder);

    }
```

```
package food.delivery.infra;

import javax.naming.NameParser;

import javax.naming.NameParser;
import javax.transaction.Transactional;

import food.delivery.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;
import food.delivery.domain.*;

@Service
@Transactional
public class PolicyHandler{
    @Autowired OrderRepository orderRepository;
    
    @StreamListener(KafkaProcessor.INPUT)
    public void whatever(@Payload String eventString){}

    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='OrderAccepted'")
    public void wheneverOrderAccepted_UpdateStatus(@Payload OrderAccepted orderAccepted){

        OrderAccepted event = orderAccepted;
        System.out.println("\n\n##### listener UpdateStatus : " + orderAccepted + "\n\n");
        // Sample Logic //
        Order.updateStatus(event);
    }
    
    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='OrderRejected'")
    public void wheneverOrderRejected_UpdateStatus(@Payload OrderRejected orderRejected){
        OrderRejected event = orderRejected;
        System.out.println("\n\n##### listener UpdateStatus : " + orderRejected + "\n\n");
        // Sample Logic //
        Order.updateStatus(event);
    }
    
    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='CookStarted'")
    public void wheneverCookStarted_UpdateStatus(@Payload CookStarted cookStarted){
        CookStarted event = cookStarted;
        System.out.println("\n\n##### listener UpdateStatus : " + cookStarted + "\n\n");
        // Sample Logic //
        Order.updateStatus(event);
    }
    
    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='DeliveryStarted'")
    public void wheneverDeliveryStarted_UpdateStatus(@Payload DeliveryStarted deliveryStarted){
        DeliveryStarted event = deliveryStarted;
        System.out.println("\n\n##### listener UpdateStatus : " + deliveryStarted + "\n\n");
        // Sample Logic //
        Order.updateStatus(event);
    }
    
    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='DeliveryFinished'")
    public void wheneverDeliveryFinished_UpdateStatus(@Payload DeliveryFinished deliveryFinished){
        DeliveryFinished event = deliveryFinished;
        System.out.println("\n\n##### listener UpdateStatus : " + deliveryFinished + "\n\n");
        // Sample Logic //
        Order.updateStatus(event);
    }
    
    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='CookFinished'")
    public void wheneverCookFinished_UpdateStatus(@Payload CookFinished cookFinished){

        CookFinished event = cookFinished;
        System.out.println("\n\n##### listener UpdateStatus : " + cookFinished + "\n\n");
        // Sample Logic //
        Order.updateStatus(event);
    }
}


```
## CQRS
![image](https://user-images.githubusercontent.com/85150301/212533981-7fcc22a4-a1d5-4053-9bf2-4cec2c99c980.png)
![image](https://user-images.githubusercontent.com/85150301/212533986-4c768795-93c7-48d2-b54d-0a91da3d06e8.png)


## Compensation / Correlation
### StoreOrder.java
```
package food.delivery.domain;

import food.delivery.domain.*;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="storeOrders", path="storeOrders")
public interface StoreOrderRepository extends PagingAndSortingRepository<StoreOrder, Long>{
    java.util.Optional<StoreOrder> findByOrderId(String orderId);
}

```

### StoreOrderRepository.java
```
public static void orderCancel(OrderCancelled orderCancelled){
        
    repository().findById(orderCancelled.getOrderId()).ifPresent(storeOrder->{
        storeOrder.setDeliveryStatus("주문 취소");  // do something
            repository().save(storeOrder);
        });
    }
}
```

