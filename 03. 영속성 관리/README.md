# 03. 영속성 관리

## 3.1. 엔티티 매니저 팩토리와 앤티티 매니저

### 엔티티 매니저 팩토리 생성 - 비용이 아주 많이 듦

```java
EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("jpabook");
```

- `Persistence.createEntityManagerFactory("jpabook")` 호출 시 META-INF/persistence.xml에 있는 정보를 바탕으로 EntityManagerFactory 생성한다.
- 생성 비용이 많이 들기 때문에 한 개만 만들어 애플리케이션 전체에서 공유한다.
- **여러 스레드 동시 접근해도 안전하므로 서로 다른 스레드 간에 공유 가능하다.**

### 엔티티 매니저 생성 - 비용 거의 들지 않음

```java
EntityManager entityManager = entityManagerFactory.createEntityManager();
```

- **여러 스레드 동시 접근 시 동시성 문제 발생하므로 스레드 간 절대 공유하면 불가능하다.**

<img width="635" alt="스크린샷 2023-05-01 오전 12 51 06" src="https://user-images.githubusercontent.com/62989828/235362907-5b5ff53e-754d-4bc4-b88d-b9f5b3d2b310.png">

엔터티 매니저는 데이터베이스 연결이 꼭 필요한 시점까지 커넥션을 얻지 않는다.
(보통 트랜잭션 시작 시 커넥션 획득)

## 3.2. 영속성 컨텍스트란?

영속성 컨텍스트(persistence context): 엔티티 영구 저장 환경, 엔티티 매니저로 엔티티를 저장하거나 조회 시 엔티티 매니저는 영속성 컨텍스트에 엔티티를 보관하고 관리

```java
entityManager.persist(member);
```

이 코드에서 `persist()` 메소드는 **엔티티 매니저를 사용해 회원 엔티티를 영속성 컨텍스트에 저장한다.**

영속성 컨텍스트는 엔티티 매니저 생성 시 하나 생성되며, 엔티티 매니저를 통해 영속성 컨텍스트에 접근 및 관리 가능하다.

(여러 엔티티 매니저가 같은 영속성 컨텍스트에 접근 가능)

## 3.3. 엔티티 생명주기

### 엔티티 상태

<img width="613" alt="스크린샷 2023-05-01 오전 1 03 30" src="https://user-images.githubusercontent.com/62989828/235363498-83abf156-2378-4e2c-a524-2c0d2528e876.png">

1. 비영속(new/transient): 영속성 컨텍스트와 전혀 관계가 없는 상태
2. 영속(managed): 영속성 컨텍스트에 저장된 상태
3. 준영속(detached): 영속성 컨텍스트에 저장되었다가 분리된 상태
4. 삭제(removed): 삭제된 상태

### 비영속

엔티티 객체 생성 시 영속성 컨텍스트나 데이터베이스와 전혀 관련이 없는 비영속 상태다.

```java
Member member = new Member();
member.setId("member1");
member.setUsername("박성우");
```

![스크린샷 2023-05-01 오후 10 46 22](https://user-images.githubusercontent.com/62989828/235460540-8d43ed8d-f87d-4380-8d8c-b73690addfec.png)

### 영속

엔티티 매니저를 통해 엔티티를 영속성 컨텍스트에 저장하여 영속성 컨텍스트가 관리하는 엔티티를 영속 상태라 한다.

```java
entityManager.persist(member);
```

<img width="602" alt="스크린샷 2023-05-01 오후 10 51 37" src="https://user-images.githubusercontent.com/62989828/235461330-507754b0-1011-4c58-b2c6-e0929df3450d.png">

### 준영속

영속성 컨텍스트가 관리하던 영속 상태의 엔티티를 영속성 컨텍스트가 관리하지 않으면 준영속 상태다.

`detach()`를 호출하거나 `close()`를 호출해 영속성 컨텍스트를 닫거나 `claer()` 를 호출해 영속성 컨텍스트 초기화 시 영속성 컨텍스트가 관리하던 영속 상태의 엔티티는 준영속 상태가 된다.

```java
entityManager.detach(member);
```

### 삭제

엔티티를 영속성 컨텍스트와 데이터베이스에서 삭제한다.

```java
entityManager.remove(member);
```

## 3.4. 영속성 컨텍스트의 특징

### 엔티티를 식별자 값으로 구분

- 식별자 값 = `@Id`로 테이블의 기본 키와 매핑한 값
- 영속 상태는 식별자 값이 반드시 있어야 함(식별자 값 없을 시 예외 발생)

### 트랜잭션 커밋 시 영속성 컨텍스트에 새로 저장된 엔티티를 데이터베이스에 반영

= 플러시(flush)

### 영속성 컨텍스트가 엔티티 관리 시 장점

1. 1차 캐시
2. 동일성 보장
3. 트랜잭션을 지원하는 쓰기 지연
4. 변경 감지
5. 지연 로딩

### 엔티티 조회

영속성 컨텍스트는 내부에 캐시(**1차 캐시**)를 가지고 있어 영속 상태의 엔티티는 모두 이곳에 저장된다.

```java
// 엔티티 생성(영속)
Member member = new Member();
member.setId("member1");
member.setUsername("박성우");

// 엔티티 영속
entityManager.persist(member);
```

<img width="602" alt="스크린샷 2023-05-01 오후 11 03 26" src="https://user-images.githubusercontent.com/62989828/235463296-30314455-c88d-4068-bbeb-fed966c2d8b5.png">

1차 캐시 키 = 식별자 값(데이터베이스 기본 키와 매핑)

∴ 영속성 컨텍스트의 데이터 저장 및 조회 기준은 **데이터베이스 기본 키 값**

```java
// 엔티티 조회
Member member = entityManager.find(Member.class, "박성우");
```

`entityManager.find()` 호출 시 먼저 1차 캐시에서 엔티티를 찾고, 찾는 엔티티가 1차 캐시에 없으면 데이터베이스에서 조회한다.

#### 1차 캐시에서 조회

`entityManager.find()` 호출 시 1차 캐시에서 식별자 값으로 엔티티를 찾는데, 찾는 엔티티가 있으면 **메모리**에 있는 1차 캐시에서 엔티티를 조회한다.

<img width="592" alt="스크린샷 2023-05-01 오후 11 11 38" src="https://user-images.githubusercontent.com/62989828/235464531-696d1eda-3d54-4e3c-b73a-0aea50f33b2f.png">

```java
Member member = new Member();
member.setId("member1");
member.setUsername("박성우");

// 1차 캐시에 저장
entityManager.persist(member);

// 1차 캐시에서 조회
Member member = entityManager.find(Member.class, "박성우");
```

#### 데이터베이스에서 조회

`entityManager.find()` 호출 시 엔티티가 1차 캐시에 없으면 엔티티 매니저는 데이터베이스를 조회해서 엔티티를 생성해 1차 캐시에 저장한 후 영속 상태의 엔티티를 반환한다.

```java
Member member2 = entityManager.find(Member.class, "박찬호");
```

<img width="597" alt="스크린샷 2023-05-01 오후 11 14 42" src="https://user-images.githubusercontent.com/62989828/235465231-add17a69-b7e5-4b10-8961-318a15c0cfb5.png">

#### 영속성 엔티티의 동일성 보장

```java
Member member1 = entityManager.find(Member.class, "박성우");
Member member2 = entityManager.find(Member.class, "박성우");

System.out.println(member1 == member2); // 동일성 비교
```

`entityManager.find(Member.class, "박성우")`를 반복 호출해도 영속성 컨텍스트는 1차 캐시에 있는 같은 엔티티를 반환하므로 `member1 == member2`의 결과는 참이다.

∴ **영속성 컨텍스트는 성능상 이점과 엔티티의 동일성을 보장한다.**

> JPA는 1차 캐시를 통해 반복 가능한 읽기(REPEATABLE READ) 등급의 트랜잭션 격리 수준을 데이터베이스가 아닌 애플리케이션 차원에서 제공한다는 장점이 있다.

### 엔티티 등록

```java
EntityManager entityManager = entityManagerFactory.createEntityManager();
EntityTransaction transaction = entityManager.getTransaction();

// 엔티티 매니저는 데이터 변경 시 트랜잭션을 시작해야 한다.
transaction.begin();    // 트랜잭션 시작

entityManager.persist(박성우);
entityManager.persist(박찬호);

// ------ INSERT SQL을 데이터베이스에 보내지 않는다. ------

transaction.commit();   // 트랜잭션 커밋

// ------ 커밋하는 순간 데이터베이스에 INSERT SQL을 보낸다. ------
```

엔티티 매니저는 트랜잭션을 커밋하기 직전까지 데이터베이스에 엔티티를 저장하지 않고 내부 쿼리 저장소에 INSERT SQL을 모아두고, 트랜잭션을 커밋할 대 모아둔 쿼리를 데이터베이스에 보낸다. = 트랜잭션을 지원하는 쓰기 지연(transactional write-behind)

<img width="604" alt="스크린샷 2023-05-01 오후 11 25 29" src="https://user-images.githubusercontent.com/62989828/235466943-c501e178-6985-4407-81d3-e5e83319529b.png">

회원 A를 먼저 영속화하면 영속성 컨텍스트는 1차 캐시에 회원 엔티티를 저장하면서 동시에 회원 엔티티 정보로 등록 쿼리를 생성해 쓰기 지연 SQL 저장소에 보관한다.

<img width="610" alt="스크린샷 2023-05-01 오후 11 25 53" src="https://user-images.githubusercontent.com/62989828/235467007-526c089c-ba39-4e90-bc2e-5d6d7001cc4e.png">

회원 B를 영속화하면 회원 엔티티 정보로 등록 쿼리를 생성해 쓰기 지연 SQL 저장소에 보관한다.

<img width="595" alt="스크린샷 2023-05-01 오후 11 27 30" src="https://user-images.githubusercontent.com/62989828/235467308-17697c85-5464-40a6-8865-2e42a29f2525.png">

트랜잭션 커밋 시 엔티티 매니저는 영속성 컨텍스트를 플러시한다.

플러시는 영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화하는 작업으로 등록, 수정, 삭제한 엔티티를 데이터베이스에 반영한다.

영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화한 후 실제 데이터베이스 트랜잭션을 커밋한다.

#### 트랜잭션을 지원하는 쓰기 지연이 가능한 이유

```java
begin();    // 트랜잭션 시작

save(강민성);
save(조윤호);
save(어정윤);

commit();   // 트랜잭션 커밋
```

1. 데이터 저장 즉시 등록 쿼리를 데이터베이스에 보내고, 마지막에 트랜잭션 커밋
2. 데이터 저장 시 등록 쿼리를 데이터베이스에 보내지 않고 메모리아 모아두었다 트랜잭션 커밋 시 모아둔 등록 쿼리를 데이터베이스에 전송 후 커밋

위 2가지 경우 모두 트랜잭션 범위 안에서 실행되므로 결과는 같다. 강민성, 조윤호, 어정윤 모두 트랜잭션 커밋 시 저장되고, 롤백 시 저장되지 않는다.

그때 그때 데이터베이스에 전달해도 트랜잭션을 커밋하지 않으면 실제 데이터베이스에는 반영되지 않기 때문에 커밋 직전에만 데이터베이스에 SQL을 전달하면 된다.

### 엔티티 수정

#### SQL 수정 쿼리의 문제점

SQL 사용 시 수정 쿼리를 직접 작성해야 한다.

회원의 이름과 나이 변경 기능 개발 후 회원 등급 변경 기능이 추가되면 보통 2개의 수정 쿼리를 작성한다.

2개의 수정 쿼리를 합쳐서 하나의 수정 쿼리만 사용해도 되지만 실수로 정보가 누락되면 원치 않은 수정이 발생한다.

이런 상황을 피하기 위해 수정 쿼리를 상황에 따라 계속 추가하면 **수정 쿼리가 많아지고, 비즈니스 로직 분석을 위해 SQL을 계속 확인해야 하므로 직접적이든 간접적이든 비즈니스 로직이 SQL에 의존하게 된다.**

#### 변경 감지

```java
EntityManager entityManager = entityManagerFactory.createEntityManager();
EntityTransaction transaction = entityManager.getTransaction();

transaction.begin();    // 트랜잭션 시작

// 영속 엔티티 조회
Member 박성우 = entityManager.find(Member.class, "박성우");

// 영속 엔티티 데이터 수정
박성우.setAge(23);
 
// entityManager.update(박성우);   // 이런 코드가 있어야 하지 않을까?

transaction.commit();   // 트랜잭션 커밋
```

JPA로 엔티티 수정 시 단순히 엔티티를 조회해서 데이터만 변경하면 된다.

엔티티의 변경사항을 데이터베이스에 자동으로 반영하는 기능을 **변경 감지(dirty checking)** 라고 한다.

<img width="591" alt="스크린샷 2023-05-01 오후 11 48 47" src="https://user-images.githubusercontent.com/62989828/235470781-d3322df4-12d9-4335-a457-de0041ec9176.png">

JPA는 엔티티를 영속성 컨텍스트에 보관 시 최초 상태를 복사해 저장(**스냅샷**)해두고, 플러시 시점에 스냅샷과 엔티티를 비교해 변경된 엔티티를 찾는다.

**변경 감지는 영속성 컨텍스트가 관리하는 영속 상태의 엔티티에만 적용된다.**

JPA의 기본 전략은 **엔티티의 모든 필드를 업데이트한다.**

수정 시 모든 필드를 사용하면 데이터베이스에 보내는 데이터 전송량이 증가하는 단점이 있지만, 아래 장점으로 인해 모든 필드를 업데이트한다.

1. 모든 필드를 사용하면 수정 쿼리가 항상 같기 때문에(바인딩 데이터는 다름) 애플리케이션 로딩 시점에 수정 쿼리를 미리 생성해두고 재사용 가능하다.
2. 데이터베이스에 동일한 쿼리를 보내면 데이터베이스는 이전에 한 번 파싱된 쿼리를 재사용 가능하다.

필드가 많거나 저장되는 내용이 너무 크면 수정된 데이터만 사용해 동적으로 UPDATE SQL을 생성하는 전략 선택(하이버네이트 확장 기능 사용 해야 함. @org.hibernate.annotations.DynamicUpdate)

> 상황에 따라 다르지만 컬럼이 약 30개 이상이면 기정적 수정 쿼리보다 동적 수정 쿼리가 빠르다.
> 컬럼이 30개 이상일 경우 테이블 설계상 책임이 적절히 분리되지 않았는지 확인해보는 것이 좋다.

### 엔티티 삭제

```java
Member 박성우 = entityManager.find(Member.class, "박성우");
entityManager.remove(박성우);  // 엔티티 삭제
```

엔티티를 삭제하려면 먼저 삭제 대상 엔티티를 조회해야 한다.

`entityManager.remove()`에 삭제 대상 엔티티를 넘겨주면 삭제 쿼리를 쓰기 지연 SQL 저장소에 등록하고, 이후 트랜잭션을 커밋해 플러시 호출 시 실제 데이터베이스에 쿼리를 전달한다.

`entityManager.remove(박성우)` 호출 시 영속성 컨텍스트에서 제거된다.

## 3.5. 플러시

**플러시(`flush()`)는 영속성 컨텍스트의 변경 내용을 데이터베이스에 반영한다.**

플러시 실행 시

1. 변경 감지가 동작해 영속성 컨텍스트에 있는 모든 엔티티를 스냅샷과 비교해 수정된 엔티티를 찾아 수정된 엔티티는 수정 쿼리를 만들어 쓰기 지연 SQL 저장소에 등록
2. 쓰기 지연 SQL 저장소의 쿼리를 데이터베이스에 전송(등록, 수정, 삭제 쿼리)

영속성 컨텍스트를 플러시하는 방법

1. `entityManager.flush()` 직접 호출
   
    - 엔티티 매니저의 `flush()` 메소드를 직접 호출해 영속성 컨텍스트를 강제로 플러시
    - 테스트나 다른 프레임워크 + JPA 사용 제외 거의 사용하지 않음

2. 트랜잭션 커밋 시 플러시 자동 호출
   
    - 데이터베이스에 변경 내용을 SQL로 전달하지 않고 트랜잭션만 커밋 시 데이터베이스에 반영되지 않는 문제 예방을 위해 JPA는 커밋할 때 플러시를 자동으로 호출
   
3. JPQL 쿼리 실행 시 플러시 자동 호출

    - JPQL이나 Criteria 같은 객체지향 쿼리 호출 시 플러시 실행
   
      JPQL쿼리 실행 시 플러시 자동 호출 이유
      
      ```java
      entityManager.persist(강민성);
      entityManager.persist(조윤호);
      entityManager.persist(어정윤);
      
      // 중간에 JPQL 실행
      query = entityManager.createQuery("select m from Member m", Member.class);
      List<Member> members = query.getResultList();
      ```
      
      `entityManager.persist()` 호출해서 강민성, 조윤호, 어정윤을 영속 상태로 만들었다.
   
      이 엔티티들은 영속성 컨텍스트에는 있지만 데이터베이스에는 반영되지 않다. 이때 JPQL을 실행하면 JPQL은 SQL로 변환되어 데이터베이스에서 엔티티를 조회하지만 강민성, 조윤호, 어정윤은 아직 데이터베이스에 없으므로 쿼리 결과로 조회되지 않는다.
   
      따라서 쿼리 실행 직전 영속성 컨텍스트를 플러시해서 변경 내용을 데이터베이스에 반영해야 하는 문제를 예방하기 위해 JPQL 실행 시에도 플러시를 자동 호출한다.
      
      (식별자를 기준으로 조회하는 `find()` 메소드 호출 시에는 플러시 실행되지 않음)
   
### 플러시 모드 옵션

엔티티 매니저에 플러시 모드 직접 지정 시 `javax.persistence.FlushModeType` 사용

1. `FlushModeType.AUTO`: 커밋이나 쿼리 실행 시 플러시(default)
2. `FlushModeType.COMMIT`: 커밋 시에만 플러시

```java
entityManager.setFlushMode(FlushModeType.COMMIT);   // 플러시 모드 직접 설정
```

## 3.6. 준영속

영속성 컨텍스트가 관리하는 영속 상태의 엔티티가 영속성 컨텍스트에서 분리된(detached) 것을 준영속 상태라 한다.

∴ **준영속 상태의 엔티티는 영속성 컨텍스트가 제공하는 기능을 사용할 수 없다.**

준영속 상태로 만드는 방법

1. `entityManager.detach(entity)`: 특정 엔티티만 준영속 상태로 전환
2. `entityManager.clear()`: 영속성 컨텍스트를 완전히 초기화
3. `entityManager.close()`: 영속성 컨텍스트 종료

### 엔티티를 준영속 상태로 전환: detach()

```java
// 회원 엔티티 생성, 비영속 상태
Member 박성우 = new Member();
박성우.setId("member1");
박성우.setUsername("박성우");

// 회원 엔티티 영속 상태
entityManager.persist(박성우);

// 회원 엔티티를 영속성 컨텍스트에서 분리, 준영속 상태
entityManager.detach(박성우);

transaction.commit();   // 트랜잭션 커밋
```

`entityManager.detach(박성우)`
 호출 시 1차 캐시, 쓰기 지연 SQL 저장소까지 해당 엔티티를 관리하기 위한 모든 정보가 제거된다.

<img width="607" alt="스크린샷 2023-05-02 오전 1 23 49" src="https://user-images.githubusercontent.com/62989828/235487123-55d1d3fc-4568-4b48-aacd-97a5c8c41578.png">

<img width="587" alt="스크린샷 2023-05-02 오전 1 25 18" src="https://user-images.githubusercontent.com/62989828/235487338-00c26435-47c5-4049-9e1f-8e55cc3e523b.png">

이렇듯 **영속 상태였다가 더는 영속성 컨텍스트가 관리하지 않는 상태를 준영속 상태**라고 한다.

### 영속성 컨텍스트 초기화: clear()

```java
// 엔티티 조회, 영속 상태
Member 박성우 = entityManager.find(Member.class, "박성우");

entityManager.clear();  // 영속성 컨텍스트 초기화

// 준영속 상태
박성우.setAge(23);
```

<img width="604" alt="스크린샷 2023-05-02 오전 1 30 23" src="https://user-images.githubusercontent.com/62989828/235488138-30eb3cf0-6fa0-46c9-a2a5-c79e387c7d06.png">

<img width="604" alt="스크린샷 2023-05-02 오전 1 30 38" src="https://user-images.githubusercontent.com/62989828/235488178-eaad4503-df88-4b8c-b624-208037477a50.png">

`entityManager.clear()` 호출로 영속성 컨텍스트 내부가 전부 초기화되어 memberA와 memberB는 더이상 영속성 컨텍스트가 관리하지 않으므로 준영속 상태다.

이때 `박성우.setAge(23)`을 호출해도 해당 엔티티가 준영속 상태이므로 영속성 컨텍스트가 지원하는 변경 감지가 동작하지 않고, 변경된 데이터가 데이터베이스에 반영되지 않는다.

### 영속성 컨텍스트 종료: close()

```java
EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("jpabook");

EntityManager entityManager = EntityManagerFactory.createEntityManager();
EntityTransaction transaction = EntityManager.getTransacction();

transaction.begin();    // 트랜잭션 시작

Member 박성우 = entityManager.find(Member.class, "박성우");
Member 박찬호 = entityManager.find(Member.class, "박찬호");

transaction.commit();   // 트랜잭션 커밋

entityManager.close();  // 영속성 컨텍스트 닫기(종료)
```

<img width="597" alt="스크린샷 2023-05-02 오전 1 37 40" src="https://user-images.githubusercontent.com/62989828/235489235-a04b318f-f04d-4444-895f-9dd9e25ecd59.png">

<img width="593" alt="스크린샷 2023-05-02 오전 1 38 00" src="https://user-images.githubusercontent.com/62989828/235489282-d64c8ec4-91af-40d4-bded-ad686ca53201.png">

영속성 컨텍스트가 종료돼 더는 memberA와 memberB가 관리되지 않는다.

> 영속 상태의 엔티티는 주로 영속성 컨텍스트가 종료되면서 준영속 상태가 된다.

### 준영속 상태의 특징

1. 거의 비영속 상태에 가깝다.
   
   영속성 컨텍스트가 관리하지 않아 1차 캐시, 씍 지연, 변경 감지, 지연 로딩을 포함한 영속성 컨텍스트가 제공하는 모든 기능 동작하지 않는다.

2. 식별자 값을 가지고 있다.

   비영속 상태는 식별자 값이 없을 수 있지만 준영속 상태는 이미 한 번 영속 상태였으므로 반드시 식별자 값을 가지고 있다.

3. 지연 로딩을 할 수 없다.

   지연 로딩(LAZY LOADING)은 실제 객체 대신 프록시 객체를 로딩해두고 해당 객체를 실제 사용할 때 영속성 컨텍스트를 통해 데이터를 불러오는 방법으로, 준영속 상태는 영속성 컨텍스트가 더는 관리하지 않으므로 지연 로딩 시 문제가 발생한다.

### 병합: merge()

준영속 상태의 엔티티를 다시 영속 상태로 변경하기 위해 병합을 사용하면 된다. `merge()` 메소드는 준영속 상태의 엔티티를 받아 그 정보로 **새로운 영속 상태의 엔티티를 반환**한다.

#### 준영속 병합

```java
public class MergeTest { 
    
    static EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("jpabook");

    public static void main(String[] args) {
        Member member = createMember("memberA", "박성우");   // (1)
        member.setUsername("박찬호");    // (2) 준영속 상태에서 변경
        mergeMember(member);  // (3)
    }
   
    static Member createMember(String id, String username) {
        // 영속성 컨텍스트1 시작
        EntityManager entityManager1 = entityManagerFactory.createEntityManager();
        EntityTransaction transaction1 = entityManager1.getTransaction();
        transaction1.begin();
      
        Member member = new Member();
        member.setId(id);
        member.setUsername(username);
      
        entityManager1.persist(member);
        transaction1.commit();
      
        entityManager1.close();   // 영속성 컨텍스트1 종료, member 엔티티는 준영속 상태가 된다.
        // 영속성 컨텍스트1 종료
      
        return member;
    }

    static void mergeMember(Member member) {
        // 영속성 컨텍스트2 시작
        EntityManager entityManager2 = entityManagerFactory.createEntityManager();
        EntityTransaction transaction2 = entityManager2.getTransaction();
        transaction2.begin();

        Member mergeMember = entityManager2.merge(member);

        transaction2.commit();

        // 준영속 상태
        System.out.println("member = " + member.getUsername());

        // 영속 상태
        System.out.println("mergeMember = " + mergeMember.getUsername());
       
        System.out.println("entityManager2 contains member = " + entityManager2.contains(member));
        System.out.println("entityManager2 contains member = " + entityManager2.contains(mergeMember));

        entityManager2.close();
        // 영속성 컨텍스트2 종료
    }
}
```

| 출력 결과

```text
member = 박찬호
mergeMember = 박찬호
entityManager2 contains member = false
entityManager2 contains mergeMember = true
```

(1) member 엔티티는 `createMember()` 메소드의 영속성 컨텍스트1에서 영속 상태였다가 영속성 컨텍스트1이 종료되며 준영속 상태가 되었으므로, `createMember()` 메소드는 준영속 상태의 member 엔티티를 반환

(2) `member.setUsername("박찬호")`을 호출해 회원 이름을 변경했지만 준영속 상태인 member 엔티티를 관리하는 영속성 컨텍스트가 존재하지 않으므로 수정 사항을 데이터베이스에 반영할 수 없다.

(3) 준영속 상태의 엔티티 수정을 위해 다시 영속 상태로 변경하는 데 병합(`merge()`)을 사용한다. `mergeMember()` 메소드에서 새로운 영속성 컨텍스트2를 시작하고 `entityManager.merge(member)`를 호출해 새로운 영속 상태의 엔티티를 반환받은 후 트랜잭션을 커밋할 때 수정 사항이 데이터베이스에 반영된다.

<img width="615" alt="스크린샷 2023-05-02 오전 2 08 48" src="https://user-images.githubusercontent.com/62989828/235494129-f7611d2c-df95-45ff-90dd-8820f462d0f0.png">

준영속 상태인 member 엔티티와 영속 상태인 mergeMember 엔티티는 서로 다른 인스턴스이다. 준영속 상태인 member는 더이상 사용할 필요가 없으므로 준영속 엔티티를 참조하던 변수를 영속 엔티티를 참조하도록 변경하는 것이 안전하다.

```java
// Member mergeMember = entityManager2.merge(member);   // 아래 코드로 변경
member = entityManager2.merge(member);
```

#### 비영속 병합

병합은 비영속 엔티티도 영속 상태로 만들 수 있다.

```java
Member member = new Member();
Member newMember = entityManager.merge(member); // 비영속 병합
transaction.commit();
```

병합 파라미터로 넘어온 엔티티의 식별자 값으로 영속성 컨텍스트를 조회하고 찾는 엔티티가 없으면 데이터베이스에서 조회하고, 데이터베이스에서도 발견하지 못하면 새로운 엔티티를 생성해 병합한다.

병합은 식별자 값으로 엔티티를 조회할 수 있으면 불러서 병합하고, 조회할 수 없으면 새로 생성해서 병합하므로, 병합은 save or update 기능을 수행한다.