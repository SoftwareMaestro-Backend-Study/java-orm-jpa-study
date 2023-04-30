# 04. 엔티티 매핑

## 목차

1. @Entity
2. @Table
3. 다양한 매핑 사용
4. 데이터베이스 스키마 자동 생성
5. DDL 생성 기능
6. 기본 키 매핑
7. 필드와 컬럼 매핑: 레퍼런스
8. 정리

## 4.1 @Entity

- JPA를 사용해서 테이블과 매핑할 클래스는 **@Entity 어노테이션을 필수로 붙여야 한다**.
- name
    - JPA에서 사용할 엔티티 이름을 지정
    - 설정하지 않으면 클래스 이름을 그대로 사용
- 주의 사항
    - 기본 생성자는 필수 (파라미터가 없는 public 또는 protected)
    - final 클래스, enum, interface, inner클래스는 적용 불가
    - 저장할 필드에 final을 사용하면 안된다.

<aside>
💡 - 엔티티를 JPA 구현체가 생성할 때 리플렉션을 사용해서 객체를 먼저 생성하고, 나중에 값을 필드에 직접 넣어줌 (기본 생성자 필요!)
- 지연로딩 등을 위해 프록시 기술을 사용

이렇게 다양한 방식으로 JPA 구현체들이 사용할 수 있도록 JPA는 스펙상 final을 사용하지 못하도록 막아두고 기본 생성자가 필요한 것이다.

</aside>

## 4.2 @Table

- 엔티티와 매핑할 테이블을 지정
- name
    - 매핑할 테이블 이름
    - 설정하지 않으면 엔티티 이름을 사용

![image](https://user-images.githubusercontent.com/83508073/235299989-6ef10191-5e5d-4990-9958-e39f5de4215e.png)


## 4.3 다양한 매핑 사용

예제로 사용할 코드는 아래와 같다.

```java
@Entity
public class Member {

    @Id
    private Long id;

    @Column(name = "name")
    private String username;

    private Integer age;

    @Enumerated(EnumType.STRING)
    private RoleType roleType;

    @Temporal(TemporalType.TIMESTAMP)
    private Date createdDate;

    @Temporal(TemporalType.TIMESTAMP)
    private Date lastModifiedDate;

    @Lob
    private String description;

    @Transient
    private int temp;
    
    protected Member(){

    }
}
```

- @Enumerated : 자바의 enum 타입 매핑
- @Temporal : 자바의 날짜 타입 매핑
- @Lob : 데이터베이스의 CLOB, BLOB 타입 매핑

## 4.4 데이터베이스 스키마 자동 생성

JPA는 데이터베이스 스키마를 자동으로 생성하는 기능을 제공한다.

JPA는 클래스의 매핑 정보와 데이터베이스 방언을 사용하여 데이터베이스 스키마를 생성한다.스프링을 사용하는 경우 아래와 같이 설정하면 된다.

- application.yml

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create
    # show-sql: true # DDL 출력
```

- create로 설정하면 기존 테이블을 삭제하고 다시 생성한다
- 자동 생성되는 DDL은 지정한 데이터베이스 방언에 따라 달라진다
- **운영 환경에서는 절대 `create`, `create-drop`, `update`를 사용하면 안된다.**
![image](https://user-images.githubusercontent.com/83508073/235300000-750cb70a-ff5c-433f-afea-5856a6fa45d8.png)


- 테스트 DB 권장 - update, validate
- 운영 DB 권장 - none

## 4.5 DDL 생성 기능

만약 테이블 컬럼에 제약 조건이 추가된다면 매핑할 엔티티에서는 스키마 자동 생성하기를 통해 만들어지는 DDL에 해당 제약 조건을 추가해야 한다. 전체적인 내용은 4.7에서 정리한다.

아래는 위 예제 코드에서 추가된 코드이다.

```java
@Entity
@Table(name="MEMER", uniqueConstraints = {@UniqueConstraint(
	name = "NAME_AGE_UNIQUE",
	columnNames = {"NAME", "AGE"})})
public class Member {

	@Id
	@Column(name = "ID")
	private String id;

	@Column(name = "NAME", nullable = false, length = 10)
	private String username;

	...
}
```

- @Column 매핑 정보에 nullable 속성 값을 false로 지정하면 자동 생성되는 DDL에 not null 제약 조건을 추가할 수 있다.
- length 속성 값을 사용하면 자동 생성되는 DDL에 문자의 크기를 지정할 수 있다.
- @Table의 uniqueConstraints을 설정하면 유니크 제약 조건을 만들어 준다.

<aside>
💡 이러한 속성들은 단지 DDL을 자동 생성할 때만 사용되고 JPA의 실행 로직에는 영향을 주지 않는다.

</aside>

## 4.6 기본 키 매핑

위 예제에서는 @Id 어노테이션만 사용하여 Member의 기본 키를 애플리케이션에서 직접 할당했다. 기본 키를 데이터베이스가 생성하주는 값을 사용하려면 어떻게 매핑해야 할까? 

- Ex) Mysql의 AUTO_INCREMENT

**데이터베이스마다 기본 키를 생성하는 방식이 서로 다르므로 JPA는 다양한 키 생성 전략을 제공한다.**

- 직접 할당: 기본 키 생성을 데이터베이스에 위임한다.
- 자동 생성: 대리 키 사용 방식
    - **IDENTITY**: 기본 키 생성을 데이터베이스에 위임한다.
    - **SEQUENCE**: 데이터베이스 시퀀스를 사용해서 기본 키를 할당한다.
    - **TABLE**: 키 생성 테이블을 사용한다.
    

### 4.6.1 기본 키 직접 할당 전략

기본 키를 직접할당하려면 @Id로 매핑하면 된다.

```java
	@Id
	@Column(name = "ID")
	private String id;
```

- 적용 가능 자바 타입
    - 자바 기본형, Wrapper형
    - String, java.util.Date, java.sql.Date, java.math.BihDecimal, java.math.BigInteger
- em.persist()로 엔티티를 저장하기 전에 애플리케이션에서 기본 키를 직접 할당해야 한다.
    
    ```java
    Board board : new Board();
    board.setid("id1); 
    em.persist(board);
    ```
    

### 4.6.2 IDENTITY 전략

기본 키 생성을 데이터베이스에 위임하는 전략이다. ex) Mysql의 **AUTO_INCREMENT**

- 데이터베이스에 값을 저장하고 나서야 기본 키 값을 구할 수 있을 때 사용한다.
    - 데이터를 데이터베이스에 Insert한 후에 기본 키 값을 조회할 수 있다.
- @GeneratedValue의 strategy 속성 값을 GenerationType.IDENTITY로 지정한다.
    - 이 전략을 사용하면 JPA는 기본 키 값을 얻어오기 위해 데이터베이스를 추가로 조회함
    - JDBC3에 추가된 Statement.**getFeneratedKeys**()를 사용하면 데이터를 저장하면서 동시에 생성된 기본 키 값도 얻어올 수 있다.
        - **하이버네이트는 이 메소드를 사용해서 데이터베이스와 1번만 통신한다.**
        
- 엔티티가 영속 상태가 되려면 식별자가 반드시 필요하다. 하지만 IDENTITY 전략은 엔티티를 데이터베이스에 저장해야 식별자를 구할 수 있으므로 em.persist()를 호출하는 즉시 Insert Sql이 데이터베이스에 전달된다.

<aside>
💡 따라서, **IDENTITY 전략은 트랜잭션을 지원하는 쓰기 지연이 동작하지 않는다.**

</aside>

### 4.6.3 SEQUENCE 전략

데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트이다.

SEQUENCE 전략은 이 시퀀스를 사용하여 기본 키를 생성한다.

```java
CREATE SEQUENCE Board_seq START WITH 1 INCREMENT BY 1;
```

```java
@Entity
@SequenceGenerator (
	name = "BOARD_SEQ_GENERATOR",
	sequenceName : "BOARD_SEQ", //매핑할 데이터베이스 시퀀스 이름 
	initialValue = 1, allocationSize : 1)
public class Board {
	@Id
	@GeneratedValue ( strategy : GenerationType.SEQUENCE,
										generator = "BOARD_SEQ_GENERATOR")
	private Long id;
	...
}
```

- @SequenceGenerator를 사용해서 시퀀스 생성기를 등록한다.
    
    ![image](https://user-images.githubusercontent.com/83508073/235356415-afc02c3d-ea57-49e5-ae43-611bdebc61ce.png)

    
- 키 전략은 GenerationType.SEQUENCE로 지정하고 등록한 시퀀스 생성기를 연결한다.
- IDENETITY와 내부 동작 방식이 다르다.
    - em.persist()를 호출할 때 먼저 데이터베이스 시퀀스를 사용해서 식별자를 조회한다.
    - 그리고 조회한 식별자를 엔티티에 할당한 후에 엔티티를 영속성 컨텍스트에 저장한다.
    - 이후 트랜잭션을 커밋해서 플러시가 일어나면 엔티티를 데이터베이스에 저장한다.

### 4.6.4 TABLE 전략

- 키 생성 전용 테이블을 하나 만들고 여기에 이름과 값으로 사용할 컬럼을 만들어서 데이터베이스 시퀀스를 흉내내는 전략이다.
- 값을 조회하면서 SELECT 쿼리를 사용하고 다음 값으로 증가시키기 위해 UPDATE 쿼리를 사용한다.
- 모든 데이터베이스에 적용할 수 있다.
- 하지만 성능이 떨어진다.

```java
create table MY_SEQUENCES ( 
 sequence_name varchar(255) not null, 
 next_val bigint, // 키 생성기를 사용할 때마다 증가
 primary key ( sequence_name )
)
```

```java
@Entity 
@TableGenerator( 
 name = "MEMBER_SEQ_GENERATOR", 
 table = "MY_SEQUENCES", 
 pkColumnValue = “MEMBER_SEQ", allocationSize = 1) 
public class Member { 
 @Id 
 @GeneratedValue(strategy = GenerationType.TABLE, 
 generator = "MEMBER_SEQ_GENERATOR") 
 private Long id;
}
```

- **@TableGenerator**

![image](https://user-images.githubusercontent.com/83508073/235356386-d7d29acf-df59-4bcc-b515-0019486ea219.png)


### 4.6.5 AUTO 전략

선택한 데이터베이스 방언에따라 IDENTITY, SEQUENCE, TABLE 전략 중 하나를 자동으로 선택한다.

- 데이터베이스를 변경해도 코드를 수정할 필요가 없다.

### 4.6.6 기본 키 매핑 정리

<aside>
💡 영속성 컨텍스트는 엔티티를 식별자 값으로 구분하므로 **엔티티를 영속 상태로 만들려면 식별자 값이 반드시 있어야 한다.**

</aside>

em.persist()를 호출한 직후 발생하는 일

1. **직접 할당**: em.persist()를 호출하기 전에 애플리케이션에서 직접 식별자 값을 할당해야 한다. 만약 식별자 값이 없으면 예외가 발생한다.
2. **SEQUENCE**: 데이터베이스 시퀀스에서 식별자 값을 획득한 후 영속성 컨텍스트에 저장한다.
3. **TABLE**: 데이터베이스 시퀀스 생성용 테이블에서 식별자 값을 획득한 후 영속성 컨텍스트에 저장한다.
4. **IDENTITY**: 데이터베이스에 엔티티를 저장해서 식별자 값을 획득한 후 영속성 컨텍스트에 저장한다.
