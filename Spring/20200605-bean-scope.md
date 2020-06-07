# 스프링 빈 스코프
빈 스코프를 지정하는 어노테이션이다. 스프링 IoC 컨테이너에 선언한 빈이 생성되고, getBean() 메서드로 빈을 요청하거나
다른 빈에서 참조할 때 스프링은 빈 스코프에 따라 어느 빈 인스턴스를 반환할 지 결졍한다.
* singleton : IoC 컨테이너당 빈 인스턴스 하나를 생성한다.
* prototype : 요청할 때마다 빈 인스턴스를 새로 만든다.
* request : HTTP 요청당 하나의 빈 인스턴스를 생성. 웹 어플리케이션 컨텍스트에만 해당
* session : HTTP 세션당 빈 인스턴스 하나를 생성. 웹 어플리케이션 컨텍스트에만 해당
* globalSession : 전역 HTTP 세션당 빈 인스턴스 하나를 생성. 포털 애플리케이션 컨텍스트에 해당

# 예제
> 카트를 나타내는 ShoppingCart 클래스를 작성한다.
~~~java
@Component
public class ShoppingCart {
    private List<Product> items = new ArrayList<>();
    
    public void addItem(Product item) {
        items.add(item);
    }
    
    public List<Product> getItems() {
        return items;
    }
}
~~~
> 상품을 나중에 카트에 추가할 수 있게 Bean으로 설정한다.
~~~java
@Configuration
public class ShopConfiguration {
    @Bean
    public Product aaa() {
        Battery p1 = new Battery();
        p1.setName("AAA");
        p1.setPrice(2.5);
        p1.setRechargeable(true);
        return p1;
    }

    @Bean
    public Product cdrw() {
        Disc p2 = new Disc("CD-RW", 1.5);
        p2.setCapacity(700);
        return p2;
    }

    @Bean
    public Product dvdrw() {
        Disc p2 = new Disc("DVD-RW", 3.0);
        p2.setCapacity(700);
        return p2;
    }
}
~~~
> Main 클래스에서 테스트
~~~java
public class Main {
    public static void main(String[] args) throws Exception {
        ApplicationContext context = 
            new AnnotationConfigApplicationContext(ShopConfiguration.class);
    
        Product aaa = context.getBean("aaa", Product.class);
        Product cdrw = context.getBean("cdrw", Product.class);
        Product dvdrw = context.getBean("dvdrw", Product.class);
    
        ShoppingCart cart1 = context.getBean("shoppingCart", ShoppingCart.class);
        cart1.addItem(aaa);
        cart1.addItem(cdrw);
        System.out.println("Shopping cart 1 contains " + cart1.getItems());

        ShoppingCart cart2 = context.getBean("shoppingCart", ShoppingCart.class);
        cart2.addItem(dvdrw);
        System.out.println("Shopping cart 2 contains " + cart2.getItems());
    }
}
~~~
> 현재 빈 구성에서는 두 카트가 동일한 인스턴스를 공유한다.
> 스프링 기본 스코프가 singleton 이기 때문에 IoC 컨테이너당 카트 인스턴스가 한개만 생성되기 때문
~~~
Shopping cart 1 contains [AAA 2.5, CD-RW 1.5]
Shopping cart 2 contains [AAA 2.5, CD-RW 1.5, DVD-RW 3.0]
~~~

> 빈 스코프를 prototype으로 변경
~~~java
@Component
@Scope("prototype")
public class ShoppingCart { ... }
~~~
> Main 클래스 재실행 시 빈 스코프가 prototype 이므로 빈 요청마다 새 인스턴스가 생성된다.
> 그래서 두 카트는 서로 다른 인스턴스가 된다.
~~~
Shopping cart 1 contains [AAA 2.5, CD-RW 1.5]
Shopping cart 2 contains [DVD-RW 3.0]
~~~