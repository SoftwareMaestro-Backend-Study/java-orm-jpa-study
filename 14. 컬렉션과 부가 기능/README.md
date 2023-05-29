# 14. 영속성과 부가 기능

## 14.1. 컬렉션

### JPA에서 지원하는 컬렉션

- Collection
- List
- Set
- Map

### JPA에서 컬렉션 사용할 수 있는 경우

- `@OneToMany`, `@ManyToMany`를 사용해 일대다나 다대다 엔티티 관계를 매핑할 때
- `@ElementColection`을 사용해 값 타입을 하나 이상 보관할 때

### 자바 컬렉션 인터페이스 특징

- Collection: 자바가 제공하는 최상위 컬렉션. 하이버네이트는 중복을 허용하고 순서를 보장하지 않는다고 가정
- Set: 중복을 허용하지 않고 순서를 보장하지 않음
- List: 순서가 있고 순서를 보장하며 중복을 허용함
- Map: key-value 구조로 되어 있는 특수한 컬렉션

### JPA와 컬렉션

하이버네이트는 엔티티를 영속 상태로 만들 때 컬렉션 필드를 하이버네이트에서 준비한 컬렉션으로 감싸서 사용한다.

하이버네이트는 컬렉션을 효율적으로 관리하기 위해 엔티티를 영속 상태로 만들 때 원본 컬렉션을 감싸고 있는 내장 컬렉션을 생성해 이 내장 컬렉션을 사용하도록 참조를 변경한다.

하이버네이트가 제공하는 내장 컬렉션은 원본 컬렉션을 감싸고 있어서 래퍼 컬렉션으로도 부른다.

하이버네이트는 이런 특징 때문에 컬렉션 사용 시 **즉시 초기화해 사용하는 것을 권장**한다.

#### 하이버네이트 내장 컬렉션과 특징

|컬렉션 인터페이스|내장 컬렉션|중복 허용| 순서 보관|
|---|---|:---:|:---:|
|Collection, List|PersistentBag|O|X|
|Set|PersistentSet|X|X|
|List + @OrderColumn|PersistentList|O|O|

### Collection, List

Collection, List는 ArrayList로 초기화하면 되고, 같은 엔티티가 있는지 찾거나 삭제하 ㄹ때 `equals()` 메소드를 사용한다.

**Collection, List는 엔티티를 추가할 때 중복된 엔티티가 있는지 비교하지 않고 단순히 저장만 하면된다.**

**따라서 엔티티를 추가해도 지연 로딩된 컬렉션을 초기화하지 않는다.**

### Set

Set은 HashSet으로 초기화하면 되고, 중복을 허용하지 않으므로 `add()` 메소드로 객체를 추가할 때마다 `equals()` 메소드로 같은 객체가 있는지 비교한다.

HashSet은 해시 알고리즘을 사용하므로 `hashcode()` 도 함께 사용해서 비교한다.

**Set은 엔티티를 추가할 때 중복된 엔티티가 있는지 비교해야 한다. 따라서 엔티티를 추가할 때 지연 로딩된 컬렉션을 초기화한다.**

### List + @OrderColumn

List 인터페이스에 `@OrderColumn`을 추가해 순서가 있는 특수한 컬렉션으로 인식할 수 있다. (**데이터베이스에 순서 값을 저장해 조회 시 사용**)

**순서가 있는 컬렉션은 데이터베이스에 순서 값도 함께 관리한다.**

#### @OrderColumn의 단점

- `@OrderColumn`을 Board 엔티티에서 매핑하므로 Comment는 POSITION 값을 알 수 없기 때문에 Comment INSERT 시 POSITION 값이 저장되지 않는다.
  Board.comments의 위치 값을 사용해 **POSITION 값을 UPDATE하는 SQL이 추가로 발생한다.**
- List 변경 시 ㅇ연관된 많은 위치 값을 변경해야 한다.
- 중간에 POSITION 값이 없으면 조회한 List에는 null이 보관된다. 따라서 컬렉션을 순회할 때 NullPointerException이 발생한다.

> 위 같은 단점 때문에 실무에서는 `@OrderColumn`을 매핑하는 대신 개발자가 직접 POSITION 값을 관리하거나 `@OrderBy`를 사용하길 권장한다.

### @OrderBy

`@OrderBy`는 데이터베이스의 ORDER BY 절을 사용해 컬렉션을 정렬한다. 따라서 순서용 컬럼을 매핑하지 않아도 되며, 모든 컬렉션에 사용할 수 있다.

`@OrderBy`의 값은 JPQL의 order by절처럼 **엔티티 필드를 대상**으로 한다.

> 하이버네이트는 Set에 @OrderBy를 적용해 결과를 조회 시 순서를 유지하기 위해 HashSet 대신 LinkedHashSet을 내부에서 사용한다.

## 14.2. @Converter

컨버터(converter) 사용 시 엔티티의 데이터를 변환해 데이터베이스 저장할 수 있다.

컨버터 클래스는 `@Converter` 어노테이션을 사용하고 AttributeConverter 인터페이스를 구현해야 한다. 제네릭에 현재 타입과 변환할 타입을 지정해야 하고, AttributeConverter
인터페이스에서 다음 두 메소드를 구현해야 한다.

- convertToDatabaseColumn(): 엔티티의 데이터를 데이터베이스 컬럼에 저장할 데이터로 변환
- convertToEntityAttribute(): 데이터베이스에서 조회한 컬럼 데이터를 엔티티의 데이터로 변환

### 글로벌 설정

모든 변환할 타입에 컨버터를 적용하려면 `@Converter(autoApply = true)` 옵션을 적용하면 된다.

이렇게 글로벌 설정을 하면 엔티티 내부에서 `@Converter`를 지정하지 않아도 모든 변환할 타입에 대해 자동으로 컨버터가 적용된다.

#### @Converter 속성 정리

|속성|기능|기본값|
|---|---|---|
|converter|사용할 컨버터 지정||
|attributeName|컨버터를 적용할 필드 지정||
|disableConversion|글로벌 컨버터나 상속 받은 컨버터를 사용하지 않음|false|

## 14.3. 리스너

JPA 리스너 기능을 사용하면 엔티티의 생명주기에 따른 이벤트를 처리할 수 있다.

### 이벤트 종류

<img width="615" alt="스크린샷 2023-05-29 오후 7 06 06" src="https://github.com/SoftwareMaestro-Backend-Study/java-orm-jpa-study/assets/62989828/3095c323-3636-4097-b99a-bc6aa5056f5f">

1. PostLoad: 엔티티가 영속성 컨텍스트에 조회된 직후 또는 refresh를 호출한 후(2차 캐시에 저장되어 있어도 호출됨)
2. PrePersist: `persist()`  메소드를 호출해 엔티티를 영속성 컨텍스트에 관리하기 직전에 호출. 식별자 생성 전략을 사용한 경우 엔티티에 식별자는 아직 존재하지 않는다. 새로운 인스턴스를
   merge할 때도 수행
3. PreUpdate: flush나 commit을 호출해 엔티티를 데이터베이스에 수정하기 직전에 호출
4. PreRemove: `remove()` 메소드를 호출해 엔티티를 영속성 컨텍스트에서 삭제하기 직전에 호출. 또한 삭제 명령어로 영속성 전이가 일어날 때도 호출. orphanRemoval에 대해서는 flush나
   commit 시에도 호출
5. PostPersist: flush나 commit을 호출해 엔티티를 데이터베이스에 저장한 직후에 호출. 식별자가 항상 존재. 식별자 생성 전략이 IDENTITY면 식별자를 생성하기 위해 `persist()`를
   호출하면서 데이터베이스에 해당 엔티티를 저장하므로 이때는 `persist()`를 호출한 직후에 바로 PostPersist가 호출됨
6. PostUpdate: flush나 commit을 호출해서 엔티티를 데이터베이스에 수정한 직후에 호출
7. PostRemove: flush나 commit을 호출해서 엔티티를 데이터베이스에 삭제한 직후에 호출

### 이벤트 적용 위치

#### 엔티티에 직접 적용

엔티티에 이벤트가 발생할 때마다 어노테이션으로 지정한 메소드 실행

#### 별도의 리스너 등록

리스너는 대상 엔티티를 파라미터로 받을 수 있음(반환 타입은 void)

#### 기본 리스너 사용

모든 엔티티의 이벤트를 처리하려면 META-INF/orm.xml에 기본 리스너로 등록

##### 여러 리스너 등록 시 이벤트 호출 순서

1. 기본 리스너
2. 부모 클래스 리스너
3. 리스너
4. 엔티티

##### 더 세밀한 설정

- javax.persistence.ExcludeDefaultListeners: 기본 리스너 무시
- javax.persistence.ExcludeSuperclassListeners: 상위 클래스 이벤트 리스너 무시

## 14.4. 엔티티 그래프

#### 엔티티 조회 시 연관된 엔티티들을 함께 조회하는 방법

1. 글로벌 fetch 옵션을 `FetchType.EAGER`로 설정

    ```java
    @Entity
    class Order {
    
        @ManyToOne(fetch = FetchType.EAGER)
        private Member member;
        ...
    }
    ```

2. JPQL에서 페치 조인 사용

    ```sql
    select o from Order o join fetch o.member;
    ```

글로벌 fetch 옵션은 애플리케이션 전체에 영향을 주고 변경할 수 없는 단점이 있어 일반적으로 글로벌 fetch 옵션은 `FetchType.LAZY`를 사용하고, 엔티티를 조회할 때 연관된 엔티티를 함께 조회할
필요가 있으면 JPQL의 페치 조인을 사용한다.

페치 조인을 사용하면 같은 JPQL을 중복해서 작성하는 경우가 많다. JPA 2.1에 추가된 엔티티 그래프 기능을 사용하면 엔티티를 조회하는 시점에 함께 조회할 연관된 엔티티를 선택할 수 있다.

**엔티티 그래프 기능은 엔티티 조회시점에 연관된 엔티티들을 함께 조회하는 기능**이다.

### Named 엔티티 그래프

Named 엔티티 그래프는 `@NamedEntityGraph`로 정의

- name: 엔티티 그래프의 이름 정의
- attributeNodes: 함께 조회할 속성 선택(이때 `@NamedAttributeNode`를 사용하고 그 값으로 함께 조회할 속성 선택)

### entityManager.find()에서 엔티티 그래프 사용

Named 엔티티 그래프를 사용하려면 정의한 엔티티 그래프를 `entityManager.getEntityGraph("Order.withMember")`를 통해 찾아오면 된다.

엔티티 그래프는 JPA 힌트 기능을 사용해 동작하는데 힌트의 키로 javax.persistence.fetchgraph를 사용하고 힌트의 값으로 찾아온 엔티티 그래프를 사용하면 된다.

### subgraph

연관된 엔티티를 2개 이상 조회할 때 subgraph를 사용한다.

### JPQL에서 엔티티 그래프 사용

JPQL에서 엔티티 그래프를 사용하는 방법은 `setHint()` 메소드를 통해 힌트만 추가하면 된다.

### 동적 엔티티 그래프

엔티티 그래프를 동적으로 구성하려면 `createEntityGraph()` 메소드를 사용하면 된다.

### 엔티티 그래프 정리

- ROOT에서 시작

  엔티티 그래프는 항상 조회하는 엔티티의 ROOT에서 시작해야 한다.

- 이미 로딩된 엔티티

  영속성 컨텍스트에 해당 엔티티가 이미 로딩되어 있으면 엔티티 그래프가 적용되지 않는다(아직 초기화되지 않은 프록시에는 엔티티 그래프가 적용된다).

- fetchgraph, loadgraph의 차이

  javax.persistence.fetchgraph 속성은 엔티티 그래프에 서냍ㄱ한 속성만 함께 조회하는 반면에 javax.persistence.loadgraph 속성은 엔티티 그래프에 선택한 속성뿐만 아니라
  글로벌 fetch 모드가 `FetchType.EAGER`로 설정된 연관관계도 포함해서 함께 조회한다.