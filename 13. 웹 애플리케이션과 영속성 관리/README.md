# 13. 웹 애플리케이션과 영속성 관리

컨테이너가 트랜잭션과 영속성 컨텍스트를 관리해주므로 애플리케이션을 손쉽게 개발할 수 있다!

컨테이너 환경에서 JPA가 동작하는 내부 동작 방식을 이해하고, 컨테이너 환경에서 웹 애플리케이션을 개발할 때 발생할 수 있는 다양한 문제점과 해결 방안을 알아보자.

## 13.1 **트랜잭션 범위의 영속성 컨텍스트**

순수하게 J2SE 환경에서 JPA를 사용하면 개발자가 직접 엔티티 매니저를 생성하고 트랜잭션도 관리해야 한다. 하지만 스프링이나 J2EE 컨테이너 환경에서 JPA를 사용하면 컨테이너가 제공하는 전략을 따라야 한다.

### 13.1.1 **스프링 컨테이너의 기본 전략**

**스프링 컨테이너는 트랜잭션 범위의 영속성 컨텍스트 전략을 기본으로 사용한다**. 

이 전략은 트랜잭션을 시작할 때 영속성 컨텍스트를 생성하고 트랜잭션이 끝날 때 영속성 컨텍스트를 종료한다. 그리고 같은 트랜잭션 안에서는 항상 같은 영속성 컨텍스트에 접근한다.

![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/2defaf79-c76b-40bf-8d44-9845f0a65181)

**@Transactional 어노테이션이 있으면 호출한 메소드를 실행하기 직전에 스프링의 트랜잭션 AOP가 먼저 동작**한다. 스프링 트랜잭션 AOP는 대상 메소드를 호출하기 직전에 트랜잭션을 시작하고, 대상 메소드가 정상 종료되면 트랜잭션을 커밋하면서 종료한다.

![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/012603b7-a28b-41b9-bb27-2539fd6c43ca)


- **트랜잭션이 같으면 같은 영속성 컨텍스를 사용**한다.
    
    엔티티 매니저를 사용하는 A,B 코드는 모두 같은 트랜잭션 범위에 있다. 따라서 엔티티 매니저는 달라도 같은 영속성 컨텍스트를 사용한다.
    
    ![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/f6dc3f52-b7d1-4284-a779-f50ad4fe53f7)

    

- **트랜잭션이 다르면 다른 영속성 컨텍스트를 사용**한다.
    
    여러 스레드에서 동시에 요청이 와서 같은 엔티티 매니저를 사용해도 트랜잭션에 따라 접근하는 영속성 컨텍스트가 다르다. 스프링 컨테이너는 스레드마다 각각 다른 트랜잭션을 할당한다.
    
    ![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/7f79f4da-8b65-4b8b-a7ca-754281e63314)

    

---

## 13.2 **준영속 상태와 지연 로딩**

스프링이나 J2EE 컨테이너는 트랜잭션이 보통 서비스 계층에서 시작하므로 서비스 계층이 끝나는 시점에 트랜잭션이 종료되면서 영속성 컨텍스트도 함께 종료된다. 조회한 엔티티가 콘트롤러나 뷰 같은 프리젠테이션 계층에서는 준영속 상태가 된다.

따라서 아래와 같이 지연 로딩 시점에서 컨트롤러에서 준영속 상태인 엔티티를 조회하면 예외가 발생한다.

```java
class OrderController {
    public String view(Long orderId) {
        Order order = orderService.findOne(orderId);
        Member member = order.getMember();
        member.getName(); // 지연 로딩 시 예외 발생
            ... 
   }
}
```

- **준영속 상태와 변경 감지**
    
    변경 감지 기능은 영속성 컨텍스트가 살아 있는 서비스 계층 (트랙잭션 범위)까지만 동작하고 영속성 컨텍스트가 종료된 프리젠테이션 계층에서는 동작하지 않는다.
    
    이는 계층이 가지는 책임을 명확히 하기 때문에 특별히 문제되지 않는다.
    
- **준영속 상태와 지연 로딩**
    
    문제는 준영속 상태는 지연 로딩 기능이 동작하지 않는다는 것이다. 
    
    뷰를 렌더링할 때 연관된 엔티티도 함께 사용해야 하는데, 이를 지연 로딩으로 설정해서 프록시 객체로 조회했다고 가정하자. **아직 초기화하지 않은 프록시 객체를 사용하면 실제 데이터를 불려오려고 초기화를 시도한다. 하지만 준영속 상태는 영속성 컨텍스트가 없으므로 지연 로딩을 할 수 없다!  → org.hiberneate.LazyInitializationException 발생**
    
- **준영속 상태 지연 로딩 해결 전략**
    
    준영속 상태의 지연 로딩 문제를 해결하는 방법은 크게 2가지가 있다.
    
    - 뷰가 필요한 엔티티를 미리 로딩해두는 방법
    - OSIV를 사용해서 엔티티를 항상 영속 상태로 유지하는 방법
    
    뷰가 필요한 엔티티를 미리 로딩해두는 방법은 어디서 미리 로딩하느냐에 따라 3가지 방법이 있다.
    
    - 글로벌 페치 전략 수정
    - JPQL 페치 조인(fetch join)
    - 강제로 초기화

### 13.2.1 **글로벌 페치 전략 수정 - 즉시 로딩**

글로벌 페치 전략을 지연 로딩에서 즉시 로딩으로 변경하면 된다.

```java
@Entitypublic class Order {
    @Id @GeneratedValueprivate Long id;

    @ManyToOne(fetch = FetchType.EAGER) // 즉시 로딩 전략
    private Member member; // 주문 회원
        ...
}
```

**글로벌 페치 전략에 로딩 사용 시 단점**

- **사용하지 않는 엔티티를 로딩한다.**
    
    화면 B에서는 order 엔티티만 있으면 충분하지만 즉시 로딩 전략으로 인해 필요하지 않은 member도 함께 조회하게 된다.
    
- **N+1 문제가 발생한다.**
    
    JPA가 JPQL을 분석해서 SQL을 생성할 때는 글로벌 페치 전략을 참고하지 않고 오직 JPQL 자체만 사용한다. 따라서 즉시 로딩 / 지연 로딩을 구분하지 않고 JPQL 쿼리 자체에 충실하게 SQL을 만든다.
    
    ```java
    List<Order> orders =
             em.createQuery("select o from Order o", Order.class)
            .getResultList(); // 연관된 모든 엔티티를 조회한다.
    // 결과
    // select * from Order
    // JPQL로 실행된 SQL
    // select * from Member where id=?
    // EAGER로 실행된 SQL
    // select * from Member where id=? 
    // EAGER로 실행된 SQL
    // select * from Member where id=? 
    // EAGER로 실행된 SQL
    // select * from Member where id=? 
    // EAGER로 실행된 SQL
    // select * from Member where id=? // EAGER로 실행된 SQL...
    ```
    

코드를 분석하면 내부에서 다음과 같은 순서로 동작한다.

1. `select o from Order o` JPQL을 분석해서 `select * from Order` SQL을 생성한다.
2. 데이터베이스에서 결과를 받아 order 엔티티 인스턴스들을 생성한다.
3. Order.member의 글로벌 페치 전략이 즉시 로딩이므로 order를 로딩하는 즉시 연관된 member도 로딩해야 한다.
4. **연관된 member를 영속성 컨텍스트에서 찾는다.**
5. **만약 영속성 컨텍스트에 없으면 `SELECT * FROM MEMBER WHERE id=?` SQL을 조회한 order 엔티티 수만큼 실행한다.**

이로 인해 order 엔티티가 10개이면 member를 조회하는 SQL도 10번 실행한다.

### 13.2.2 **JPQL 페치 조인**

방금 설명한 N+1 문제가 발생했던 예제에서 JPQL만 fetch join을 사용하도록 변경하자.

join 뒤에 fetch를 넣어주면 된다. 이로써 SQL JOIN을 사용해서 페치 조인 대상까지 함께 조회하여 N+1 문제가 발생하지 않는다.

```java
JPQL:
    select o
    from Order o
    join **fetch** o.member
SQL:
    select o.*, m.*
    from Order o
    join Member m on o.MEMBER_ID = m.MEMBER_ID
```

- **JPQL 페치 조인의 단점**
    
    페치 조인이 현실적인 대안이긴 하지만 무분별하게 사용하면 화면에 맞춘 리포지터리 메소드가 증가할 수 있다.
    
    - 화면 A를 위해 order만 조회하는 repository.findOrder() 메소드
    - 화면 B를 위해 order와 연관된 member를 페치 조인으로 조회하는 `repository.findOrderWithMember()` 메소드
    
    이제 화면 A와 화면 B에 각각 필요한 메소드를 호출하면 된다. 이처럼 메소드를 각각 만들면 최적화는 할 수 있지만 뷰와 리포지터리 간에 논리적인 의존관계가 발생한다. 무분별한 최적화로 프리젠테이션 계층과 데이터 접근 계층 간에 의존과계가 급격하게 증가하는 것보다는 적절한 선에서 타협점을 찾는 것이 합리적이다.
    

### 13.2.3 **강제로 초기화**

영속성 컨텍스트가 살아있을 때 프리젠테이션 계층이 필요한 엔티티를 **강제로 초기화**해서 반환하는 방법이다.

```java
class OrderService {
    @Transactionalpublic Order findOrder(id) {
        Order order = orderRepository.findOrder(id);
        order.getMember().getName(); // 프록시 객체를 강제로 초기화한다.
        return order;
    }
}
```

하이버네이트를 사용하면 **initialize() 메소드**를 사용해서 프록시를 강제로 초기화할 수 있다.

```java
org.hibernate.Hibernate.initialize(order.getMember()); // 프록시 초기화
```

이처럼 프록시를 초기화하는 역할을 서비스 계층이 담당하면 뷰가 필요한 엔티티에 따라 서비스 계층의 로직을 변경해야 한다.

이는 **프리젠테이션 계층이 서비스 계층을 침범하는 상황**이다. 따라서 비즈니스 로직을 담당하는 서비스 계층에서 프리젠테이션 계층을 위한 프록시 초기화 역할을 분리해야 한다. 

**FACADE 계층**이 그 역할을 담당해 줄 것이다.

### 13.2.4 **FACADE 계층 추가**

프리젠테이션 계층과 서비스 계층 사이에 FACADE 계층을 하나 더 두는 방법이다. **덕분에 서비스 계층은 프리젠테이션 계층을 위해 프록시를 초기화 하지 않아도 된다.** 결과적으로 논리적인 의존성을 분리할 수 있다.

![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/68f70b4e-a042-4525-876e-ddf58a365fbf)

프록시를 초기화하려면 영속성 컨텍스트가 필요하므로 FACADE에서 트랜잭션을 시작해야 한다.

- **FACADE 계층의 역할과 특징**
    - 프리젠테이션 계층과 도메인 모델 계층 간의 논리적 의존성을 분리해준다.
    - 프리젠테이션 계층에서 필요한 프록시 객체를 초기화한다.
    - 서비스 계층을 호출해서 비즈니스 로직을 실행한다.
    - 리포지터리를 직접 호출해서 뷰가 요구하는 엔티티를 찾는다.

```java
class OrderFacade {
    @Autowired
    OrderService orderService;
    public Order findOrder(id) {
        Order order = orderService.findOrder(id);
        //프리젠테이션 계층에 필요한 프록시 객체를 강제로 초기화한다.
        order.getMember().getName();
        return order;    
    }
}

class OrderService {
    public Order findOrder(id) {
        return orderRepository.findOrder(id);
    }
}
```

이제 서비스 계층은 비즈니스 로직에 집중하고 프리젠테이션 계층을 위한 초기화 코드는 모두 FACADE가 담당하면 된다. 하지만 실용적인 관점에서 볼 때 FACADE의 최대 단점은 중간에 계층이 하나 더 끼어든다는 점이다.

### 13.2.5 **준영속 상태와 지연 로딩의 문제점**

뷰를 개발할 때 필요한 엔티티를 미리 초기화하는 방법은 새악ㄱ보다 오류가 발생할 가능성이 높다.

또한 FACADE를 이용해서 준영속 상태의 지연로딩 문제를 어느 정도 해소할 수는 있지만 상당히 번거롭다. 예를 들어 주문 엔티티와 연관된 회원 엔티티를 조회할 때 화면별로 최적화된 엔티티를 딱딱 맞아 떨어지게 초기화해서 조회하려면 FACEDE 계층에 여러 종류의 조회 메소드가 필요하다.

결국 모든 문제는 엔티티가 프리젠테이션 계층에서 준영속 상태이기 때문에 발생한다.

따라서 영속성 컨텍스트를 뷰까지 살아있게 열어두자. 그럼 뷰에서도 지연 로딩을 사용할 수 있는데 이것이 OSIV다.

## 13.3 **OSIV**

OSIV(Open Session In View)는 영속성 컨텍스트를 뷰까지 열어둔다는 뜻이다.

따라서 뷰까지 엔티티가 영속 상태로 유지되고, 지연 로딩을 사용할 수 있다.

### 13.3.1 **과거 OSIV: 요청 당 트랜잭션**

OSIV의 핵심은 뷰에서도 지연 로딩이 가능하도록 하는 것이다.

![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/268844c7-b20a-4d18-b68e-f1c964274e44)

그림과 같이 요**청이 들어오자마자 서블릿 필터나 스프링 인터셉터에서 영속성 컨텍스트를 만들면서 트랜잭션을 시작하고 요청이 끝날 때 트랜잭션과 영속성 컨텍스트를 함께 종료**한다.

이로써 영속성 컨텍스트가 처음부터 끝까지 살아있으므로 조회한 엔티티도 영속 상태를 유지한다. 

 

- **요청 당 트랜잭션 방식의 OSIV 문제점**
    
    요청 당 트랜잭션 방식의 OSIV가 가지는 문제점은 컨트롤러나 뷰 같은 프리젠테이션 계층이 엔티티를 변경할 수 있다는 점이다. 프리젠테이션 계층에서 엔티티를 수정하지 못하게 막는 방법들은 다음과 같다.
    
    - 엔티티를 읽기 전용 인터페이스로 제공
    - 엔티티 래핑
    - DTO만 반환
- 1. **엔티티를 읽기 전용 인터페이스로 제공**
    
    엔티티를 직접 노출하는 대신에 다음 예제와 같이 읽기 전용 메소드만 제공하는 인터페이스를 프리젠테이션 계층에 제공하는 방법이다.
    
    ```java
    interface MemberView {
        public String getName();
    }
    
    @Entityclass Member implements MemberView {
        ...
    }
    
    class MemberService {
        public MemberView getMember(id) {
            return memberRepository.findById(id);
        }
    }
    ```
    

- 2. **엔티티 래핑**
    
    엔티티의 읽기 전용 메소드만 가지고 있는 엔티티를 감싼 객체를 만들고 이것을 프리젠테이션 계층에 반환하는 방법이다.
    
    ```java
    class MemberWarpper {
        private Member member;
        public MemberWrapper(member) {
            this.member = member;
        }
        //읽기 전용 메소드만 제공
        public String getName() {
            return member.getName();
        }
    }
    ```
    

- 3. **DTO만 반환**
    
    가장 전통적인 방법으로 프리젠테이션 계층에 엔티티 대신에 단순히 데이터만 전달하는 객체인 DTO를 생성해서 반환하는 것이다. 하지만 이 방법은 OSIV를 사용하는 장점을 살릴 수 없고 엔티티를 거의 복사한 듯한 DTO 클래스도 하나 더 만들어야 한다.
    

지금까지 설명한 OSIV는 요청 당 트랜잭션 방식의 OSIV다. 최근에는 이런 문제점을 어느정도 보완해서 비즈니스 계층에서만 트랜잭션을 유지하는 방식의 OSIV를 사용한다. 스프링 프레임워크가 제공하는 OSIV가 바로 이 방식을 사용하는 OSIV다.

## 13.3.2 **스프링 OSIV: 비즈니스 계층 트랜잭션**

OSIV를 서블릿 필터에서 적용할지 스프링 인터셉터에서 적용할지에 따라 원하는 클래스를 선택해서 사용하면 된다. 

- 서블릿 필터에 적용: `OpenEntityManagerInViewFilter`를 서블릿 필터에 등록
- 스프링 인터셉터에 적용: `OpenEntityManagerInViewInterceptor`를 스프링 인터셉터에 등록

- **스프링 OSIV 분석**
    
    스프링 프레임워크가 제공하는 OSIV는 비즈니스 계층에서 트랜잭션을 사용하는 OSIV다.
    
    ![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/b5a778ef-81d5-48c7-9e98-ac8725847e39)
    
    1. 클라이언트의 요청이 들어오면 서블릿 필터나, 스프링 인터셉터에서 **영속성 컨텍스트를 생성**한다. 단 **이때 트랜잭션은 시작하지는 않는다.**
    2. 서비스 계층에서 `@Transactional`로 트랜잭션을 시작할 때 **1번에서 미리 생성해둔 영속성 컨텍스트를 찾아와서 트랜잭션을 시작한다.**
    3. 서비스 계층이 끝나면 **트랜잭션을 커밋하고 영속성 컨텍스트를 플러시**한다. 이때 트랜잭션은 끝내지만 영속성 컨텍스트는 종료하지 않는다.
    4. 컨트롤러와 뷰까지 영속성 컨텍스트가 유지되므로 조회한 엔티티는 영속 상태를 유지한다.
    5. **서블릿 필터나, 스프링 인터셉터로 요청이 들어오면 영속성 컨텍스트를 종료한다. 이때 플러시를 호출하지 않고 바로 종료한다.**

- **트랜잭션 없이 읽기**
    
    엔티티를 변경하지 않고 단순히 조회만 할 때는 트랜잭션이 없어도 되는데 이것을 **트랜잭션 없이 읽기**라 한다. 프록시를 초기화하는 지연 로딩도 조회 기능이므로 트랜잭션 없이 읽기가 가능하다.
    
    - 영속성 컨텍스트는 트랜잭션 범위 안에서 엔티티를 조회하고 수정할 수 있다.
    - 영속성 컨텍스트는 트랜잭션 범위 밖에서 엔티티를 조회만 할 수 있다. 이것을 트랜잭션 없이 읽기 (Nontranscational reads)라 한다.

- **OSIV 특징 정리**
    - 영속성 컨텍스트를 프리젠테이션 계층까지 유지한다.
    - 프리젠테이션 계층에는 트랜잭션이 없으므로 엔티티를 수정할 수 없다.
    - 프리젠테이션 계층에는 트랜잭션에 없지만 트랜잭션 없이 읽기를 사용해서 지연로딩을 할 수 있다.

- **컨트롤러에선 플러시가 동작하지 않는 이유는 무엇일까?**
    
    컨트롤러에서 회원 엔티티를 member.setName(”XXX”)로 변경한다고 가정하자. 프리젠테이션 계층이지만 아직 영속성 컨텍스트가 살아있고, 만약 영속성 컨텍스트를 플러시하면 변경 감지가 동작해서 데이터베이스에 변경된 이름을 반영할 것이다. 하지만 다음과 같은 2가지 이유 덕분에 컨트롤러에서는 플러시가 동작하지 않는다. 
    
    - 트랜잭션을 사용하는 서비스 계층이 끝날 때 트랜잭션이 커밋되면서 이미 플러시해버렸다. 스프링이 제공하는 OSIV 서블릿 필터나 OSIV 스프링 인터셉터는 요청이 끝나면 플러시를 호출하지 않고 `em.close()`로 영속성 컨텍스트만 종료해 버리므로 플러시가 일어나지 않는다 .
    - 프리젠테이션 계층에서 `em.flush()`를 호출해서 강제로 플러시해도 트랜잭션 범위 밖이이므로 데이터를 수정할 수 없다는 예외를 만난다.
    
    따라서 컨트롤러에서 영속 상태의 엔티티를 수정했지만, 수정 내용이 데이터베이스에는 반영되지 않는다. 
    

- **스프링 OSIV 주의사항**
    
    프리젠티이션 계층에서 엔티티를 수정해도 수정 내용이 데이터베이스에 반영되는 예외가 있다. 프리젠테이션 계층에서 엔티티를 수정한 직후에 트랜잭션을 시작하는 서비스 계층을 호출하면 문제가 발생한다.
    
    ```java
    class MemberController {
        public String viewMember(Long id) {
            Member member = memberService.getMember(id);
            member.setName("XXX"); // 보안상의 이유로 고객 이름을 XXX로 변경했다.
            memberService.biz(); // 비즈니스 로직
            return "view";
        }
    }
    ```
    
    ![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/73f7fa32-6eca-4e8a-8a15-97963c6f1a8e)
    
    위의 코드는 biz() 메소드가 끝나면 **트랜잭션 AOP는 트랜잭션을 커밋하고 영속성 컨텍스트를 플러시한다.** 이때 **변경 감지가 동작하면서 회원 엔티티의 수정 사항을 데이터베이스에 반영**한다.
    
    이런 문제를 해결하는 단순한 방법은 트랜잭션이 있는 비즈니스 로직을 모두 호출하고 나서 엔티티를 변경하면 된다.
    
    ```java
    class MemberController {
        public String viewMember(Long id) {
            memberService.biz(); // 비즈니스 로직
        
            Member member = memberService.getMember(id);
            member.setName("XXX"); // 보안상의 이유로 고객 이름을 XXX로 변경했다.
    				return "view";
        }
    }
    ```
    

스프링 OSIV는 같은 영속성 컨텍스트를 여러 트랜잭션이 공유할 수 있으므로 이런 문제가 발생한다. OSIV를 사용하지 않는 트랜잭션 범위의 영속성 컨테스트 전략은 트랜잭션의 생명주기와 영속성 컨텍스트의 생명주기가 같으므로 이런 문제가 발생하지 않는다.

### 13.3.3 **OSIV 정리**

- **스프링 OSIV의 특징**
    - 한 번 조회한 엔티티는 요청이 끝날 때까지 영속 상태를 유지한다.
    - 엔티티 수정은 트랜잭션이 있는 게층에서만 동작한다.
- **스프링 OSIV의 단점**
    - OSIV를 적용하면 같은 영속성 컨텍스트를 여러 트랜잭션이 공유할 수 있다는 점을 주의해야 한다.
    - 프리젠테이션 계층에서 엔티티를 수정하고나서 비즈니스 로직을 수행하면 엔티티가 수정될 수 있다.
    - 프리젠테이션 계층에서 지연 로딩에 의한 SQL이 실행된다. 따라서 성능 튜닝시에 확인해야 할 부분이 넓다.
- **OSIV vs FACADE vs DTO**
    
    OSIV를 사용하지 않는 대안은 FACADE 계층이나 그것을 조금 변형해서 사용하는 방법이 있는데 어떤 방법을 사용하든 준영속 상태가 되기 전에 프록시를 초기화해야 하는 단점이 있다.
    
- **OSIV를 사용하는 방법이 만능은 아니다**
    
    OSIV를 사용하면 화면을 출력할 때 엔티리를 유지하면서 객체 그래프를 마음껏 탐색할 수 있다. 하지만 복잡한 화면을 구성할 때는 이 방법이 효과적이지 않은 경우가 많다. 엔티티를 직접 조회하기보다는 JPQL로 필요한 데이터들만 조회해서 DTO로 반환하는 것이 더 나은 해결책일 수 있다.
    
- **OSIV는 같은 JVM을 벗어난 원격 상황에서는 사용할 수 없다**
    
    OSIV는 같은 JVM을 벗어난 원격 상황에서는 사용할 수 없다. JSON이나 XML을 생성할 때는 지연 로딩을 사용할 수 있지만 원격지인 클라이언트에서 연관된 엔티티를 지연 로딩하는 것은 불가능하다. 보통 Jackson이나 Gson 같은 라이브러리를 사용해서 객체를 JSON으로 변환하는데, 변환 대상 객체로 엔티티를 직접 노출하거나 또는 DTO를 사용해서 노출한다.
    
    이렇게 JSON으로 생성한 API는 한 번 정의하면 수정하기 어려운 외부 API와 언제든지 수정할 수 있는 내부 API로 나눌 수 있다.
    

## 13.4 너무 엄격한 계층

아래 코드에서는 상품을 구매한 후 구매 결과 엔티티를 조회하려고 컨트롤러에서 레포지토리를 직접 접근한다.

```java
class OrderController {
	@Autowired OrderService orderService;
	@Autowired OrderRepository orderRepository;

	public String orderRequest(Order order, Model model) {
		long Id = orderService.order(order);
		
		// 레포지토리 직접 접근
		Order orderResult = orderRepository.findOne(id);
		model.addAtrribute("order", orderResult);
		...
	}
} 
```

OSIV를 사용하기 전에는 프리젠테이션 계층에서 사용할 지연 로딩된 엔티티를 미리 초기화해야 했다. 하지만 OSIV를 사용하면 영속성 컨텍스트가 프리젠테이션 계층까지 살아있으므로 미리 초기화할 필요가 없다. 따라서 단순한 엔티티 조회는 컨트롤러에서 직접 호출해도 문제 없다.

![image](https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/83508073/b936fcaf-0676-452e-a9e2-7003e3555fb8)

## 13.5 **정리**

JPA를 사용하면 트랜잭션이라는 단위로 영속성 컨텍스트를 관리하므로 트랜잭션을 커밋하거나 롤백할 때 문제가 없다. 유일한 단점은 프리젠테이션 계층에서 엔티티가 준영속 상태가 되므로 지연 로딩을 할 수 없다는 점이다.

기존 OSIV는 프리젠테이션 계층에서도 엔티티를 수정할 수 있다는 단점이 있었다. 스프링 프레임워크가 제공하는 OSIV는 기존 OSIV의 단점들을 해결해서 프리젠테이션 계층에서 엔티티를 수정하지 않는다.
