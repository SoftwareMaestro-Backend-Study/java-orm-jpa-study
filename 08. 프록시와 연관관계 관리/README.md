# 08. 프록시와 연관관계 관리

객체는 객체 그래프를 통해 연관된 객체들을 탐색한다.  하지만 객체가 데이터베이스에 저장되어 있으면 연관된 객체를 마음껏 탐색하기 어렵다. 

따라서 JPA 구현체들은 이 문제를 해결하기 위해서 프록시 기술을 사용한다. 

프록시를 통해서 연관된 객체를 처음부터 데이터베이스에서 조회하지 않고, 실제 사용하는 시점에 데이터베이스에서 조회할 수 있다. (지연 로딩)

하지만, 자주 함께 사용되는 객체들은 조인을 통해 함께 조회하는 것이 효과적이다. (즉시 로딩)

따라서  JPA는 위 두 가지를 **지연 로딩**과 **즉시 로딩**으로 모두 지원한다.

## 8.1 프록시

엔티티를 조회할 때 항상 연관된 엔티티들이 사용되지는 않는다.

```java
// CASE 1. Member, Team 객체 조회 필요
public void printUserAndTeam(String memberId) {
	Member member = em.find(Member.class, memberId);
	Team team = member.getTeam();
	System.out.println("회원 이름: " + member.getUsername());
	System.out.println("소식팀: " + team.getName()); // team 객체 조회
}

// CASE 2. Member 객체 조회 필요
public void printUser(String memberId) {
	Member member = em.find(Member.class, memberId);
	Team team = member.getTeam();
	System.out.println("회원 이름: " + member.getUsername());
}
```

Case2의 경우 회원 엔티티만 출력하기 때문에 회원과 연관된 엔티티는 전혀 사용하지 않는다.

따라서, printUser() 메소드는 em.find()로 회원 엔티티를 조회할 때 연관된 엔티티인 팀까지 데이터베이스에서 함께 조회해 두는 것은 효율적이지 않다.

따라서 JPA에서는 엔티티가 실제 사용될 때까지 데이터베이스 조회를 지연하는 **지연 로딩**을 제공한다. 이를 통해 팀 엔티티의 값을 실제 사용하는 시점에서 데이터베이스에서 조회한다. 

이때, 실제 팀 엔티티 대신에 데이터베이스 조회를 지연할 수 있는 가짜 객체가 필요한데, 이것을 **프록시 객체**라 한다.

<aside>
💡 JPA는 지연 로딩 구현 방법을 JPA 구현체에 위임하였다. 따라서 아래의 내용은 JPA 구현체 중 하나인 하이버네이트에 대한 내용이다.  

지연 로딩을 구현하는 방법은 (1) 바이트코드를 수정하는 방법과 (2) 프록시 객체를 사용하는 방법이 있다.

</aside>

### 8.1.1 프록시 기초

**em.find() vs em.getReference()**

(1) **EntityManager.find()**를 사용하면 영속성 컨텍스트에 엔티티가 없으면 데이터베이스를 조회한다.

```java
Member member = em.find(Member.class, "member1");
```

(2) 엔티티를 실제 사용하는 시점까지 데이터베이스 조회를 미루고 싶으면 **EntityManager.getReference()** 메소드를 사용하면 된다. 

```java
Member member = em.getReference(Member.class, "member1");
```

- 이 메소드를 호출하면 데이터베이스 접근을 위임한 **프록시 객체**를 반환한다. 데이터베이스를 조회하지 않고 실제 엔티티 객체도 생성하지 않는다.

![프록시 객체를 반환한다.](https://user-images.githubusercontent.com/83508073/236809781-9d568259-f3a1-451b-a9a2-4546d09199b7.png)

프록시 객체를 반환한다.

**프록시 객체 초기화**

프록시 객체는 **실제 객체에 대한 참조(target)를 보관**한다.

프록시 객체의 메소드를 호출하면 프록시 객체는 실제 객체의 메소드를 호출한다. 이를 **프록시 객체 초기화**라 한다.

![image](https://user-images.githubusercontent.com/83508073/236809847-fb444f70-d1cc-409e-b544-af8a2be18043.png)

**프록시의 특징**

- 프록시 객체는 **처음 사용할 때 한 번만 초기화된다**.
- **프록시 객체를 초기화한다고 프록시 객체가 실제 엔티티로 바뀌는 것은 아니다.** 프록시 객체가 초기화되면 **프록시 객체를 통해서 실제 엔티티에 접근할 수 있다.**
- **프록시 객체는 원본 엔티티를 상속받은 객체**이므로 타입 체크 시에 주의해서 사용해야 한다. 또한 사용하는 입장에서는 진짜 객체와 프록시 객체를 구분하지 않고 사용하면 된다.
    
    ![image](https://user-images.githubusercontent.com/83508073/236809954-49ce1504-6961-480f-bdc7-1e5e4e482902.png)
    
- 프록시 객체는 **실제 객체에 대한 참조를 보관**한다.
그리고 프록시 객체의 메소드를 호출하면 **프록시 객체가 실제 객체의 메소드를 호출**한다.
    
    ![image](https://user-images.githubusercontent.com/83508073/236809992-5a66b06c-11e9-48e5-b99b-114a8052f67a.png)
    
- **영속성 컨텍스트에 찾는 엔티티가 이미 있으면** 데이터베이스를 조회할 필요가 없으므로 **em.getReference()를 호출해도 프록시가 아닌 실제 엔티티를 반환한다.**
- 초기화는 영속성 컨텍스트의 도움을 받아야 가능하다. 
따라서 영속성 컨텍스트의 도움을 받을 수 없는 **준영속 상태의 프록시를 초기화하면 문제가 발생**한다. 하이버네이트는 `org.hibernate.LazyInitializationException` 예외를 발생시킨다.

```java
// MemberProxy 반환
Member member = em.getReference(Member.class, "id1");

transaction.commit();
em.close(); // 영속성 컨텍스트 종료 -> 모두 준영속 상태

member.getName(); // 준영속 상태 초기화 시도 -> 예외 발생
```

### 8.1.2 프록시와 식별자

엔티티를 프록시로 조회할 때 식별자(PK) 값을 파라미터로 전달하는데, 프록시 객체는 이 식별자 값을 보관한다.

```java
Team team = em.getReference(Team.class, "team1"); // 식별자 보관
team.getId(); // 초기화되지 않음
```

- 프록시객체는 식별자값을 가지고 있으므로 식별자 값을 조회하는 team.getId()를 호출 해도 프록시를 초기화하지 않는다.
    
    단, `@Access(AccessType.PROPERTY)`로 설정한 경우에만 초기화하지 않는다.
    
- 엔티티 접근 방식을 필드 `@Access(AccessType.FIELD)`로 설정하면 JPA는 getId() 메소드가 id만 조회하는 메소드인지 다른 필드까지 활용해서 어떤 일을 하는 메소드인지 알지 못하므로 프록시 객체를 초기화한다.

### 8.1.3 프록시 확인

PersistenceUtil.isLoaded(Object entity) 메소드를 사용하면 프록시 인스턴스의 초기화 여부를 확인할 수 있다.

```java
boolean isLoad = em.getEntityManagerFactory()
										.getPersistenceUnitUtil().isLoaded(entity);
//또는 boolean isLoad = emf.getPersistenceUnitUtil().isLoaded(entity);

System.out.println("isLoad = " + isLoad); // 초기화 여부 확인
```

하이버네이트의 initialize() 메소드를 통해 프록시를 강제로 초기화할 수 있다.

```java
org.hibernate.Hibernate.initialize(order.getMember()); // 프록시 초기화
```

## 8.2 즉시 로딩과 지연 로딩

회원 엔티티를 조회할 때, 연관된 팀 엔티티를 함께 데이터베이스에서 조회하는 것이 좋을까? 아니면 회원 엔티티만 조회하는 것이 좋을까?

상황에 따라 지연 로딩과 즉시 로딩을 선택해야 한다!

- 즉시 로딩 : 엔티티를 조회할 때 연관된 엔티티도 함께 조회한다.
    - 설정 방법 : @ManyToOne(fetch = FetchType.**EAGER**)
- 지연 로딩 : 연관된 엔티티를 실제 사용할 때 조회한다.
    - 설정 방법 : @ManyToOne(getch = FetchType.**LAZY**)
    

### 8.2.1 즉시 로딩

@ManyToOne의 fetch 속성을 FetchType.EAGER로 지정한다.

```java
// 즉시 로딩 실행 코드
Member member = em.find(Member.class, "member1");
Team team = member.getTeam(); // 객체 그래프 탐색
```

![image](https://user-images.githubusercontent.com/83508073/236810068-397c65c4-7a43-4d3b-a173-a5ccb870a98c.png)

대부분의 JPA 구현체는 즉시 로딩을 최적화하기 위해 가능하면 조인 쿼리를 사용한다.

```sql
// 실제 쿼리
SELECT
    M.MEMBER_ID AS MEMBER_ID,
    M.TEAM_ID AS TEAM_ID,
    M.USERNAME AS USERNAME,
    T.TEAM_ID AS TEAM_ID,
    T.NAME AS NAME
FROM MEMBER M 
LEFT OUTER JOIN TEAM T
	  ON M.TEAM一ID=T.TEAM一ID
WHERE
    M.MEMBER_ID='member1'
```

**NULL 제약조건과 JPA 조인 전략**

현재 회원 테이블에 TEAM_ID 외래 키는 Null 값을 허용하고 있다. 따라서 팀에 소속되지 않은 회원이 있을 가능성이 있다. **팀에 소속하지 않은 회원과 팀을 내부 조인 하면 팀은 물론이고 회원 데이터도 조회할 수 없다.**

하지만 외부 조인보다 내부 조인이 성능과 최적화에서 더 유리하다.

**내부 조인을 사용하려면 어떻게 해야 할까?**

외래 키에 NOT NULL 제약 조건을 설정하면 값이 있는 것을 보장한다. NOT NULL을 표현하는 방법은 두 가지가 있다.

- @JoinColumn(name = "TEAM_ID", nullable = false) :
- @ManyToOne(fetch = FetchType.EAGER, optional = false)

<aside>
💡 정리하자면 
- **선택적 관계면 외부 조인을 사용 (nullable = true)**
- **필수 관계면 내부 조인을 사용 (nullable = false)**

</aside>

### 8.2.2 지연 로딩

@ManyToOne의 fetch 속성을 FetchType.LAZY로 지정한다.

```java
// 지연 로딩 실행 코드
Member member = em.find(Member.class, "member1");
Team team = member.getTeam(); // 프록시 객체
team.getName(); // 팀 객체 실제 사용 
```

![image](https://user-images.githubusercontent.com/83508073/236810127-5ed85bd7-f6a1-4218-8f1f-403fa1a3bcba.png)

이때, 2번째 line에서 team은 프록시 객체이고, 3번째 line에서 team 객체를 실제 사용하므로 데이터베이스 조회가 이뤄진다.

```sql
SELECT * FROM MEMBER
WHERE MEMBER_ID = "member1"

SELECT * FROM TEAM
WHERE TEAM_ID = "team1"
```

<aside>
💡 - **조회 대상이 영속성 컨텍스트에 이미 있으면 프록시 객체를 사용할 이유가 없다.** 
따라서 프록시가 아닌 실제 객체를 사용한다.

예를 들어 team1 엔티티가 영속성 컨텍스트에 이미 로딩되어 있으면 프록시가 아닌 실제 team1 엔티티를 사용한다.

</aside>
