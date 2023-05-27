# 09. 값 타입

### JPA의 데이터 타입 분류

1. 엔티티 타입

- @Entity로 정의하는 객체
- 데이터가 변해도 식별자로 지속해서 추적 가능

2. 값 타입

- int, Integer, String처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체
- 식별자가 없고 값만 있으므로 변경시 추적 불가

### 값 타입 분류

1. 기본값 타입

- 자바 기본 타입(ex. int, double)
- 래퍼 클래스(ex. Integer, Long)
- String

2. 임베디드 타입(복합 값 타입)
3. 컬렉션 값 타입

## 9.1. 기본값 타입

- 식별자 값이 없다.
- 엔티티의 생명주기에 의존한다.

  ex. 회원을 삭제 시 이름, 나이 필드도 함께 삭제

- **기본값 타입은 절대 공유하면 안된다.**
- 항상 값을 복사한다.

  Integer같은 래퍼 클래스나 String같은 특수한 클래스는 공유 가능한 객체이지만 변경하지 않는다.

## 9.2. 임베디드 타입(복합 값 타입)

- 엔티티의 생명주기에 의존한다.
- 엔티티의 비슷한 정보들을 하나로 묶어 새로운 값 타입으로 정의해서 사용한다.

  ex. 회원 엔티티에 주소 도시, 주소 번지, 주소 우편번호가 있다면 이러한 데이터는 객체지향적이지 않으며 응집력이 떨어진다. 대신 이 세 필드를 묶어 "주소"라는 새로운 타입으로 만들어준다면 코드가 훨씬 명확해
질 수 있다.
- 임베디드 타입 사용 시 **재사용성과 응집도가 높아진다.**
- **기본 생성자가 필수**다.
- 사용
  - **@Embeddable**: 값 타입을 정의하는 곳
  - **@Embedded**: 값 타입을 사용하는 곳

### 임베디드 타입과 테이블 매핑

임베디드 타입은 엔티티의 값일 뿐이다. 따라서 값이 속한 엔티티의 테이블에 매핑한다.

### 임베디드 타입과 연관관계

임베디드 타입은 값 타입을 포함하거나 엔티티를 참조할 수 있다.

### @AttributeOverride: 속성 재정의

임베디드 타입에 정의한 매핑정보를 재정의하려면 엔티티에 @AttributeOverride 어노테이션을 사용하면 된다.

예를 들어 ```Address``` 값 타입으로 집 주소와 회사 주소를 선언한다고 하면 테이블에 매핑하는 컬럼명이 중복되는 문제가 발생한다. 이런 경우 @AttributeOverrides를 사용해 매핑정보를
재정의해야 한다.

```java
@Embedded
@AttributeOverrides({
        @AttributeOverride(name = "city", column = @Column(name = "COMOPANY_CITY")),
        @AttributeOverride(name = "street", column = @Column(name = "COMPANY_STREET")),
        @AttributeOverride(name = "zipcode", column = @Column(nmae = "COMPANY_ZIPCODE"))
})
Address companyAddress;
```

> 임베디드 타입이 임베디드 타입을 가지고 있어도 @AttributeOverrides는 엔티티에 설정해야 한다.

### 임베디드 타입과 null

임베디드 타입이 null이면 매핑한 컬럼 값은 모두 null이 된다.

## 9.3. 값 타입과 불변 객체

### 값 타입 공유 참조

임베디드 타입 같은 값 타입을 여러 엔티티에서 공유하면 위험하다.

아래 예시의 경우 회원2에게 새로운 주소를 할당하기 위해 회원1의 주소를 가져와 수정한 후 회원2의 주소로 설정해주었다. 이때 address의 값을 변경할 경우 의도한 바와는 다르게 회원1과 회원2의 주소가 모두 변경된다.

```java
member1.setHomeAddress(new Address("OldCity"));
Address address = member1.getHomeAddress();

// 값 타입 공유
address.setCity("NewCity");    // 회원1의 address 값을 공유해서 사용 -> 바람직하지 않음
member2.setHomeAddress(address);
```

한 객체의 정보를 변경했을 때 다른 객체의 정보도 함께 변경되는 부작용을 막기 위해 값 타입을 복사해서 사용해야 한다.

```java
member1.setHomeAddress(new Address("OldCity"));
Address address = member1.getHomeAddress();

// 값 타입 복사
Address newAddress = address.clone();   // 회원1의 address 값을 복사해서 새로운 newAddress 값 생성

newAddress.setCity("NewCity");
member2.setHomeAddress(newAddress);
```

자바의 기본 타입은 값을 대입하면 값을 복사해서 전달한다. 하지만 **임베디드 타입처럼 직접 정의한 값타입은 객체 타입이므로 객체를 대입할 때 항상 참조 값을 전달**한다. 따라서 객체를 대입할 때마다 인스턴스를
복사해서 대입해야 한다. 문제는 복사하지 않고 원본의 참조 값을 직접 넘기는 것을 막을 방법이 없다. 따라서 이를 해결할 수 있는 가장 단순한 방법은 객체의 값을 수정하지 못하게 막는 것이다.

### 불변 객체

한 번 만들면 절대 변경할 수 없는 객체를 불변 객체라 한다. 객체를 불변하게 만들면 값을 수정할 수 없으므로 부작용을 원천 차단할 수 있다. 따라서 **값 타입은 될 수 있으면 불변 객체로 설계해야 한다.**

## 9.4. 값 타입의 비교

자바가 제공하는 객체 비교

1. 동일성(identity) 비교: 인스턴스의 참조 값을 비교, `==` 사용
2. 동등성(equivalence) 비교: 인스턴스의 값을 비교, `equals()` 사용

> 자바에서 eqauls()를 재정의할 때 hashCode()도 재정의하는 것이 안전하다.
> 그렇지 않으면 해시를 사용하는 컬렉션(HashSet, HashMap)이 정상 동작하지 않는다.

## 9.5. 값 타입 컬렉션

값 타입을 하나 이상 저장하려면 컬렉션에 보관하고 `@ElementCollection`, `@CollectionTable` 어노테이션을 사용하면 된다.

| Member.java

```java
@Entity
public class Member {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @Embedded
    private Address homeAddress;
    
    @ElementCollection
    @CollectionTable(name = "FAVORITE_FOODS", joinColumns = @JoinColumn(name = "MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private Set<String> favoriteFoods = new HashSet<>();
    
    @ElementCollection
    @CollectionTable(name = "ADDRESS", joinColumns = @JoinColumn(name = "MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private List<Address> addressHistory = new ArrayList<>();
}
```

| Address.java

```java
@Embeddable
public class Address {
    
    @Column
    private String City;
    private String street
    private String zipcode;
    ...
}
```