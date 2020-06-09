# Overview
Spring HATEOAS 프로젝트는 `HATEOAS(Hypertext as Engine of Application State)` 원칙을 따르는 REST 표현을 쉽게 작성하는데
사용할 수 있는 API 라이브러리이다.

# Dependency
~~~groovy
dependencies {
    ...

    implementation 'org.springframework.boot:spring-boot-starter-hateoas'
    ...
}
~~~

# 예제 작성
> Spring HATEOAS는 URI 생성을 위해 `RepresentationModel, Link 및 WebMvcLinkBuilder` 세 가지 추상화를 제공한다.
> 이를 사용하여 메타 데이터를 작성하고, 자원 표현에 연관시킬 수 있다.
## 1. 모델 작성
> 모델 객체는 `RepresentationModel` 기본 클래스를 상속한다.
> 향후 해당 모델에 `link`를 add 하여 쉽게 URI를 생성할 수 있다.
~~~java
@Getter
@AllArgsConstructor(staticName = "of")
@NoArgsConstructor
public class Customer extends RepresentationModel<Customer> {
    private String customerId;
    private String customerName;
    private String companyName;
}
~~~

## 2. 링크 만들기
> Controller 클래스에서 모델에 링크를 만들어 추가한다. `WebMvcLinkBuilder` 클래스를 사용하여 URI를 손쉽게 작성할 수 있다.
* linkTo() : 컨트롤러 클래스를 검사하고, 해당 루트 Path를 취득한다.
* slash() : URI 경로(PathVariable)의 변수을 지정한다.
* withSelfRel() : 관계를 자체 링크로 규정한다.
~~~java
@RestController
@RequestMapping("/customers")
@RequiredArgsConstructor
public class CustomerController {
    ...
    
    @GetMapping("/{customerId}")
    public Customer getCustomerById(@PathVariable String customerId) {
        Customer customer = customerService.getCustomerDetail(customerId);
        customer.add(linkTo(CustomerController.class)
            .slash(customer.getCustomerId())
            .withSelfRel());

        return customer;
    }
}
~~~

## 3. Relations
### 모델 작성 추가
~~~java
@Getter
@AllArgsConstructor(staticName = "of")
@NoArgsConstructor
public class Order extends RepresentationModel<Order> {
    private String orderId;
    private double price;
    private int quantity;
}
~~~

### List 형태의 모델들에 링크 만들기
> `methodOn()` 을 이용하여 컨트롤러의 메소드를 기반으로 URI 링크를 손쉽게 만들 수 있다.
~~~java
@RestController
@RequestMapping("/customers")
@RequiredArgsConstructor
public class CustomerController {
    ...

    @GetMapping(value = "/{customerId}/orders", produces = { "application/hal+json" })
        public CollectionModel<Order> getOrdersForCustomer(@PathVariable final String customerId) {
            List<Order> orders = orderService.getAllOrdersForCustomer(customerId);
            for (final Order order : orders) {
                Link selfLink = linkTo(methodOn(CustomerController.class)
                        .getOrderById(customerId, order.getOrderId()))
                        .withSelfRel();
    
                order.add(selfLink);
            }
    
            Link link = linkTo(methodOn(CustomerController.class)
                    .getOrdersForCustomer(customerId))
                    .withSelfRel();
    
            return CollectionModel.of(orders, link);
        }
}
~~~

## 4. 모든 링크 결합해보기
~~~java
@RestController
@RequestMapping("/customers")
@RequiredArgsConstructor
public class CustomerController {
    ...
    
    @GetMapping(produces = { "application/hal+json" })
    public CollectionModel<Customer> getAllCustomers() {
        List<Customer> allCustomers = customerService.allCustomers();

        for (Customer customer : allCustomers) {
            String customerId = customer.getCustomerId();
            Link selfLink = linkTo(CustomerController.class)
                    .slash(customerId)
                    .withSelfRel();

            customer.add(selfLink);

            if (orderService.getAllOrdersForCustomer(customerId).size() > 0) {
                Link ordersLink = linkTo(methodOn(CustomerController.class)
                        .getOrdersForCustomer(customerId)).withRel("allOrders");

                customer.add(ordersLink);
            }
        }

        Link link = linkTo(CustomerController.class)
                .withSelfRel();

        return CollectionModel.of(allCustomers, link);
    }
}
~~~
***호출 해보기***
~~~bash
> curl -G http://localhost:8080/customers | jq '.'
~~~
~~~json
{
  "_embedded": {
    "customerList": [
      {
        "customerId": "10A",
        "customerName": "Jane",
        "companyName": "ABC Company",
        "_links": {
          "self": {
            "href": "http://localhost:8080/customers/10A"
          },
          "allOrders": {
            "href": "http://localhost:8080/customers/10A/orders"
          }
        }
      },
      {
        "customerId": "10B",
        "customerName": "John",
        "companyName": "DEF Company",
        "_links": {
          "self": {
            "href": "http://localhost:8080/customers/10B"
          },
          "allOrders": {
            "href": "http://localhost:8080/customers/10B/orders"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:8080/customers"
    }
  }
}
~~~
전체 코드는 [hateoas-tutorial](https://github.com/leeyh0928/hateoas-tutorial)에서 확인 가능하다.

# References
* https://www.baeldung.com/spring-hateoas-tutorial